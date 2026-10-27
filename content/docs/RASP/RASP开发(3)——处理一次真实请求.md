---
title: RASP开发(3)——处理一次真实请求
date: 2026-10-23
---

# RASP开发(3)——处理一次真实请求

## 前言

在 w3 之前，我已经实现了将 Hook 链路打通，然后在这个链路上增加同步判断安全判定，让命中的命令无法执行。而在 w2 结束时暴露出两个问题：

1. **L0 知道要阻断，但是不知道当前 HTTP 请求是谁，也拿不到响应对象。**因为我们这个 RASP 是依附于 Java Web 运行的，所以 RASP 也会加载到运行 Tomcat 的同一个 JVM 中，检测代码会在处理请求的线程上执行，而当我们拦截之后抛出`SecurityBlockError`之后会被 Tomcat 接收并认为是服务器错误，所以返回 500。这就是为什么我在 w2.2 的测试中发现并没有按照设想中的 403 回显而是 500 了。
2. **审计不能在业务线程中直接写日志**。需要先非阻塞入队，再由后台线程异步交付。简单来说，就是处理 HTTP 请求的线程，只负责把审计记录放进队列，真正输出日志的工作交给后台的另一个线程了。审计记录就是此次安全事件，比如哪个请求执行了什么危险操作、命中了什么规则、是否阻断等。Tomcat 使用工作线程处理请求，当线程执行到`ProcessBuilder.start()`时，也会执行我们插入的 L0 检查。如果检查命中之后直接写日志，Tomcat 就要等待写日志和抛出阻断信号，写日志、输出到控制台、发送网络日志都可能变慢，比如磁盘忙，线程就需要等待；大量攻击同时命中规则时，可能会让很多请求线程排队等待写日志，从而拖慢业务。

那么这里**异步审计怎么做**？

我们可以在中间加一个队列：

```text
Tomcat 请求线程：
L0 命中 → 构造审计记录 → 尝试放入队列 → 继续阻断

                              队列
                               ↓
后台审计线程：
                         取出记录 → 写日志
```

这样我们的请求线程就不用等待发送日志之类的操作了。

**非阻塞入队**具体指什么？

在项目中使用的是`queue.offer(record)`。队列是由容量上限的，如果有空位，就把记录放进去，返回 true；如果队列满了，直接返回 false，同时记录丢弃指标，业务线程继续执行阻断。而如果使用`put()`方法，队列满时线程就需要等到有空位，这样会让审计系统拖住业务线程。所以**非阻塞**准确来说就是不等待队列空位，也不等待日志写完。

## W3 任务

由于 w2 已经实现能拦住命令，所以 w3 的任务就是能够正确处理一次  Web 请求，并且记录安全事件。主要是四项：

1. 建立请求上下文
   在 `HttpServlet.service()` 入口保存请求身份、request、response，通过 `ThreadLocal` 让深层的命令 Hook 找到当前请求；请求退出时清理，嵌套调用结束后恢复父上下文。
2. 把阻断正确表现为 HTTP 403
   L0 命中后，通过 Servlet 响应适配器准备 403，再抛出 `SecurityBlockError` 阻止命令执行；请求出口按条件处理这个阻断信号，避免正常阻断变成 Tomcat 500。
3. 增加异步审计
   请求线程只尝试把事件放入有界队列，由后台线程输出日志。同时处理队满计数、线程退出和 ClassLoader 引用，避免审计拖慢业务或造成资源泄漏。
4. 修复简单的命令匹配绕过
   W2 精确匹配能识别 `whoami`，却可能漏掉一个参数里的 `whoami > file`。W3 增加分隔符和大小写处理，覆盖这些简单变形。

### 请求载体 Advice

因为`ProcessBuilder.start()`本身是没有`HttpServletRequest`和`HttpServletResponse`参数的，所以我们还需要一个 Hook 来识别一次请求的开始和结束。这个 Hook 就是请求载体 Hook。

载体可以理解为业务操作所在的处理过程。对于 Servlet  Web 应用，w3 选择在`HttpServlet.service(request, response)`上 Hook，因为它是处理 HTTP 请求的一个边界。

这个 Hook 主要负责：

- 进入时：建立上下文，保存请求身份和 request、response。
- 执行期间：让内部发生的危险操作能找到当前请求。
- 退出时：按条件处理阻断信号，清理上下文，有嵌套调用就恢复父上下文。

所以一个大致的调用就是：

```text
HttpServlet.service(request, response)
    ↓ OnMethodEnter
ContextStore.enter(request, response)
    ↓ 业务代码
ProcessBuilder.start(command)
    ↓
SyncGuard.check()
    ↓ 命中
ResponseBlocker.block(ContextStore.current(), error)
    ↓ OnMethodExit
ContextStore.exit(context)
```

这里有一个很重要的 RASP 设计原则：**危险 Hook 和请求载体 Hook 是两个不同层次的 Hook**。前者负责判定危险操作，后者负责提供请求身份和响应边界。把它们分开，才能支持 Servlet、RPC、MQ 等不同载体。

### ThreadLocal 上下文的作用和边界

`ThreadLocal`是 Java 中用于创建**线程局部变量**的类，它的核心作用是让每个访问该变量的线程都用于自己**独立的一个副本**，从而避免多线程环境下的资源共享冲突。

因为我们想要知道一次请求中是否存在攻击性行为，所以我们要获得请求的相关信息，也就是要拿到 request 和 response。因为传统 Servlet 请求通常在一个工作线程内同步执行，也就是 Tomcat 通常会从线程池中选一个工作线程，让它从头到尾处理这一次请求，请求的过程中，各层的方法在这个线程上依次执行，所以如果我们想要让`ProcessBuilder.start()`找到当前请求对象，就可以用`ThreadLocal`把当前请求信息暂存在处理它的线程上，这个线程里后续执行的代码就能取到，无需通过方法参数逐层传递。

比如：

假设一个请求要经过三层业务代码，最后执行命令：

```java
// Servlet 层：这里有 request、response
service(request, response) {
    userService.query();
}

// Service 层：没有 request、response
query() {
    commandUtil.run();
}

// 工具层：也没有 request、response
run() {
    new ProcessBuilder("whoami").start();
}
```

到了 `ProcessBuilder.start()`，我们的危险 Hook 虽然知道命令是什么，却没有 `response`，因此不知道该给哪个 HTTP 响应设置 403。

一种做法是修改所有中间方法，把响应对象一路传下来：

```text
service(request, response)
    → query(request, response)
        → run(request, response)
```

但作为 Agent，我们希望直接接入已有应用，尽量不要求业务修改代码。所以 W3 使用 `ThreadLocal`。我们可以把它理解为：**每个线程都有一个专属储物格。**

```text
Tomcat 工作线程 A

进入 service(request, response)
    ↓
请求载体 Hook：
把包含 request、response 的 RequestContext 存进线程 A 的储物格
    ↓
userService.query()
    ↓
commandUtil.run()
    ↓
ProcessBuilder.start()
    ↓
危险 Hook：
从线程 A 的储物格取出 RequestContext
    ↓
取得 response，准备 403
    ↓
请求结束：清理储物格
```

**关键条件是这些代码运行在同一个线程上。** 方法调用得再深，也能取到这个线程保存的上下文；如果切换到另一个线程，普通 `ThreadLocal` 不会自动把上下文传过去。另外，Tomcat 会复用工作线程处理后续请求，所以请求结束时必须清理或恢复父上下文，否则下一个请求可能拿到上一个请求的信息。

### 阻断适配器

在 w2 中，我们已经实现了基本的拦截，也就是命中规则之后抛出`SecurityBlockError`，但是要让客户端收到 HTTP 403，我们还需要操作 Servlet 的 response，这就是适配器`BlockAdapter`的工作。

不同的服务类型就会有不同的阻断适配器，所以我们对于`BlockAdapter`接口只设计两件事：是否能够处理这种上下文以及执行这个框架对应的响应处理。而具体怎么返回 403，就看实现类怎么做了。

目前实现的是对于 Servlet 的阻断，由于 Java 的历史比较曲折，从 Sun 到 Oracle，中间还有一些版权纠纷什么的，所以 Servlet 有两个命名空间：`javax.servlet-api`和`jakarta.servlet-api`，但是其中的内容是没有变的，为了避免直接依赖这两个命名空间，我们使用反射来调用响应对象。

### 审计线程的 ClassLoader





























