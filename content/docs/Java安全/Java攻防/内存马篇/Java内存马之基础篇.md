---
title: Java 内存马之基础篇
date: 2026-10-14
---

# Java 内存马之基础篇

## Java Web 三大件

在学习 Tomcat 之前，需要先学习一下 Java Web 三大件，即 Servlet、Filter、Listener。当 Tomcat 接收到请求时，会依次经过 Listener -> Filter -> Servlet。

### Servlet

#### Servlet 简介

Java Servlet 是运行在 Web 服务器或应用服务器上的程序，它是作为来自 Web 浏览器或其他 HTTP 客户端的请求和 HTTP 服务器上的数据库或应用程序之间的中间层。

它在应用程序中一般在这个位置：

![1](https://drun1baby.top/2022/08/19/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-01-%E5%9F%BA%E7%A1%80%E5%86%85%E5%AE%B9%E5%AD%A6%E4%B9%A0/ServletLike.png)

#### 请求的处理过程

客户端发起一个 HTTP 请求，比如 GET 类型。Servlet 容器接收到请求，根据请求信息，封装`HttpServletRequest`和`HttpServletResponse`对象，这一步也就是我们的传参。Servlet 容器调用`HttpServlet`的`init()`，`init()`方法只在第一次请求的时候被调用。Servlet 容器调用`service()`方法，`service()`方法根据请求类型这里是 GET 类型，分别调用`doGet`或者`doPost`方法，这里调用`doGet`方法。`doXXX`方法中是我们自己写的业务逻辑。

业务逻辑处理完成之后，返回给 Servlet 容器，然后容器将结果返回给客户端。容器关闭时，会调用`destroy()`方法。

> 一整套的流程用代码来表示应该是这样的：

```java
package tomcatShell.Servlet;  
  
import javax.servlet.*;  
import javax.servlet.annotation.WebServlet;  
import java.io.IOException;  
  
// 基础恶意类  
@WebServlet("/servlet")  
public class ServletTest implements Servlet {  
    @Override  
 	public void init(ServletConfig config) throws ServletException {  
  
    }  
  
    @Override  
 	public ServletConfig getServletConfig() {  
        return null;  
 	}  
  
    @Override  
 	public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {  
        
    }  
  
    @Override  
 	public String getServletInfo() {  
        return null;  
 	}  
  
    @Override  
 	public void destroy() {  
  
    }  
}
```

#### Servlet 生命周期

1. 服务器启动时（web.xml 中配置`load-on-startup=1`，默认为 0）或者第一次请求该 Servlet 时，就会初始化一个 Servlet 对象，也就是会执行初始化方法`init(ServletConfig conf)`。
2. Servlet 对象去处理所有客户端请求，在`service(ServletRequest req，ServletResponse res)`方法中执行。
3. 服务器关闭时，销毁这个 Servlet 对象，执行`destroy()`方法。
4. 由 JVM 进行垃圾回收。

### Filter

#### Filter 简介

Filter 也被称为过滤器，是对 Servlet 技术的一个强补充，其主要功能是在`HttpServletRequest`在到达`Servlet`之前，拦截客户的`HttpServletRequest`，根据需要检查`HttpServletRequest`，也可以修改`HttpServletRequest`头和数据；在`HttpServletResponse`到达客户端之前，拦截`HttpServletResponse`，根据需要检查`HttpServletResponse`，也可以修改`HttpServletResponse`头和数据。

工作原理：

![1](https://drun1baby.top/2022/08/19/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-01-%E5%9F%BA%E7%A1%80%E5%86%85%E5%AE%B9%E5%AD%A6%E4%B9%A0/filterWorking.png)

不过在这里，如果我们想办法在 Filter 前自己创建一个 Filter 并且放到最前面，我们的 Filter 就会最先执行，当我们在 Filter 中添加恶意代码，就会进行命令执行，这样就成为了一个内存 Webshell。

#### 基本工作原理

1. Filter 程序是一个实现了特殊接口的 Java 类，与 Servlet 类似，也是由 Servlet 容器进行调用和执行的。
2. 当在 web.xml 注册了一个 Filter 来对某个 Servlet 程序进行拦截处理时，它可以决定是否将请求继续传递给 Servlet 程序，以及对请求和响应消息是否进行修改。
3. 当 Servlet 容器开始调用某个 Servlet 程序时，如果发现已经注册了一个 Filter 程序来对该 Servlet 进行拦截，那么容器不再调用 Servlet 的`service`方法，而是调用 Filter 的`doFilter()`方法决定是否去激活`service()`方法。
4. 但是在`Filter.doFilter()`方法中不能直接调用 Servlet 的`service()`方法，而是调用`FilterChain.doFilter()`方法来激活目标 Servlet 的`service()`方法，`FilterChain`对象是通过`Filter.doFilter()`方法的参数传递进来的。
5. 只要在`Filter.doFilter()`方法中调用`FilterChain.doFilter()`方法的语句前后增加某些程序代码，这样就可以在 Servlet 进行响应前后实现某些特殊功能。
6. 如果在`Filter.doFilter()`方法中没有调用`FilterChain.doFilter()`方法，则目标 Servlet 的`service()`方法就不会被执行，这样通过 Filter 就可以阻止某些非法的访问请求。

> 简单来说就几点：
>
> 其实就是 Filter 中的 Filter 访问需要在 web.xml 中定义路径，因为有些接口我们需要加 Filter，有些则不需要。
>
> Filter 有一条 FilterChain，也就是由多个 Filter 组成的，会一个个进行 Filter 的操作，最后一个 Filter 最后执行`Servlet.service()`。

#### Filter 的生命周期

与 Servlet 一样，Filter 的创建和销毁也由 Web 容器负责。Web 应用程序启动时，Web 服务器将创建 Filter 的实例对象，并调用其`init()`方法，读取 web.xml 配置，完成对象的初始化功能，从而为后续的用户请求做好拦截的准备工作（Filter 对象只会创建一次，`init()`方法也只会执行一次）。开发人员通过`init()`方法的参数，可获得代表当前 Filter 配置信息的`FilterConfig`对象。Filter 对象创建后会驻留在内存中，当 Web 应用移除或服务器停止时才会销毁，Web 容器卸载 Filter 对象之前被调用。该方法在 Filter 的生命周期中只执行一次。在这个方法中，可以释放过滤器使用的资源。

看一下代码：

```java
package EvilFliter;  
  
import javax.servlet.*;  
import java.io.IOException;  
  
public class FilterTest implements Filter {  
    @Override  
    public void init(FilterConfig filterConfig) throws ServletException {  
  
    }  
  
    @Override  
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {  
        // 在这里面进行 doGet 和 doPost 这种类似的  
 	}  
  
    @Override  
    public void destroy() {  
  
    }  
}
```

#### Filter 链

当多个 Filter 同时存在时，组成了 Filter 链。Web 服务器根据 Filter 在 web.xml 文件中的注册顺序，决定先调用哪个 Filter。当第一个 Filter 的`doFilter()`方法被调用时，Web 服务器会创建一个代表 Filter 链的`FilterChain`对象传递给该方法，通过判断`FilterChain`中是否还有 Filter 决定后面是否还调用 Filter。

![1](https://drun1baby.top/2022/08/19/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-01-%E5%9F%BA%E7%A1%80%E5%86%85%E5%AE%B9%E5%AD%A6%E4%B9%A0/FilterChain.png)

### Listener

#### Listener 简介

Java Web 开发中的监听器（Listener）就是 Application、Session 和 Request 三大对象创建、销毁或者往其中添加、修改、删除属性时自动执行代码的功能组件。

- `ServletContextListener`：对 Servlet 上下文的创建和销毁进行监听；
- `ServletContextAttributeListener`：监听 Servlet 上下文属性的添加、删除和替换；
- `HttpSessionListener`：对 Session 的创建和销毁进行监听。Session 的销毁有两种情况，一种是 Session 超时，另一种是通过调用 Session 对象的`invalidate()`方法使 Session 失效；
- `HttpSessionAttributeListener`：对 Session 对象中属性的添加、删除和替换进行监听；
- `ServletRequestListener`：对请求对象的初始化和销毁进行监听；
- `ServletRequestAttributeListener`：对请求对象属性的添加、删除和替换进行监听。

#### 用途

可以使用监听器监听客户端的请求、服务端的操作等。通过监听器，可以自动触发一些动作，比如监听在线的用户数量、统计网站的访问量、网站访问监控等。

## Tomcat 基础

### Tomcat 简介

Tomcat 我们大致对标 Apache 来看。

Apache 是 Web 服务器（静态解析，如 HTML），Tomcat 是 Java 应用服务器（动态解析，如 JSP）。Tomcat 只是一个 Servlet 容器，因为 JSP 也会被翻译为 Servlet，可以认为是 Apache 的扩展，但是也可以独立于 Apache 运行。简单来说，就是一个 Web 服务器，不是很稳定，但是业务能力较强。

### Tomcat 和 Servlet 的关系

我们知道 **Tomcat 是 Web 应用服务器，是一个 Servlet/JSP 容器**，而 Servlet 容器从上到下分别是 Engine、Host、Context、Wrapper。

这是 Tomcat 的容器层级结构，描述的是 Servlet 容器内部如何组织和管理 Web 应用：

```text
Engine（引擎）
 └── Host（虚拟主机）
      └── Context（Web 应用）
           └── Wrapper（单个 Servlet）
```

**Engine —— 引擎**

- 代表整个 **Servlet 容器**（一个 Tomcat 实例的 Catalina 引擎）
- 一个 Service 通常对应一个 Engine，它是**顶层容器**，负责接收请求并分发给下面的 Host
- 它不直接处理具体应用，只做**请求路由的起点**

**Host —— 虚拟主机**

- 对应一个**域名/虚拟主机**，比如`www.example.com`、`127.0.0.1`
- 一个 Engine 下可以有多个 Host，用于在一台服务器上托管多个域名站点
- 它负责根据请求的 **Host 头** 找到对应的虚拟主机

**Context —— Web 应用**

- 对应一个**部署的 Web 应用**（一个 war 包或者一个应用目录），比如`/myapp`
- 它管理这个应用自己的 Servlet、Filter、Listener、Session、类加载器等
- 在`server.xml`中常见`<Context path="/myapp" docBase="..."/>`

**Wrapper —— 单个 Servlet**

- 对应**一个具体的 Servlet**（最底层容器）
- 每个`<servlet>`配置对应一个 Wrapper，它负责该 Servlet 的加载、初始化、调用`service()`方法
- 是请求处理的**最终执行单元**

在 Tomcat 中 Wrapper 代表一个独立的 Servlet 实例，`StandardWrapper`接口是 Wrapper 接口的标准实现类（`StandardWrapper`的主要任务就是载入 Servlet 类并且进行实例化），同时其从`ContainerBase`类继承过来，表示它是一个容器，只是它是最底层的容器，不能再含有任何子容器了，且其父容器只能是 context，而我们现在也就是需要在这里去载入我们自定义的 Servlet 加载我们的内存马。

## Tomcat 架构

### Tomcat 架构原理

Tomcat 的框架如图所示：

![1](https://drun1baby.top/2022/08/19/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-01-%E5%9F%BA%E7%A1%80%E5%86%85%E5%AE%B9%E5%AD%A6%E4%B9%A0/TomcatStage.png)

这里可以看到 Tomcat 的核心组件是`Connector`和`Container`：

- `Connector`主要负责对外交流，进行 Socket 通信（基于 TCP/IP），解析 HTTP 报文，对应下图中 HTTP 服务器
- `Container`主要处理`Connector`接收的请求，主要是处理内部事务，加载和管理 Servlet，由 Servlet 具体处理 Request 请求，对应下图中的 Servlet 容器。

![1](https://drun1baby.top/2022/08/19/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-01-%E5%9F%BA%E7%A1%80%E5%86%85%E5%AE%B9%E5%AD%A6%E4%B9%A0/TomcatUse.png)

#### Server

即服务器，代表整个 Tomcat 服务器，它要能够提供一个接口让其它程序能够访问到这个 Service 集合、同时要维护它所包含的所有 Service 的生命周期，包括如何初始化、如何结束服务、如何找到别人要访问的 Service。还有其它的一些次要的任务。

一个 Tomcat 中只有一个 Server，一个 Server 中包含至少一个 Service 组件，用于提供具体服务。

#### Service

Service 主要是为了关联 Connector 和 Container，同时会初始化它下面的其他组件，在 Connector 和 Container 外面多包装了一层，把他们组装在一起，向外部提供服务，一个 Service 可以设置多个 Connector，但是只有一个 Container 容器。

Tomcat 中 Service 接口的标准实现类是 StandardService ，它不仅实现了 Service 接口同时还实现了 Lifecycle 接口，这样它就可以控制它下面的组件的生命周期了。

#### Connector

Connector 组件是 Tomcat 中两个核心组件之一，它的主要任务是负责接收浏览器发过来的 TCP 连接请求，创建一个 Request 和 Response 对象分别用于和请求端交换数据，然后会产生一个线程来处理这个请求并把产生的 Request 和 Response 对象传给处理这个请求的线程，处理这个请求的线程就是 Container 组件要做的事了。

根据运行的逻辑，我们可以发现 Connector 主要有三个功能：

> 1. socket 通信
> 2. 解析处理应用层协议，如将 socket 连接封装成 request 和 response 对象，后续交给 Container 来处理
> 3. 将 Request 转换为 ServletRequest，将 Response 转换为 ServletResponse

其中 Tomcat 设计了三个组件，其负责功能：

- EndPoint: 负责网络通信，将字节流传递给 Processor；
- Processor: 负责处理字节流生成 Tomcat Request 对象，将 Tomcat Request 对象传递给 Adapter；
- Adapter: 负责将 Tomcat Request 对象转化成 ServletRequest 对象，传递给容器。

##### Adapter 组件

由于协议的不同，Tomcat 定义了自己的 Request 类来存放请求信息，但是这个不是标准的 ServletRequest。于是需要使用 Adapter 将 Tomcat Request 对象转成 ServletRequest 对象，然后就能调用容器的`service()`方法。

简单来说，Endpoint 接收到 Socket 连接之后，生成一个 SocketProcessor 任务提交到线程池中进行处理，SocketProcessor 的`run()`方法将调用 Processor 组件进行应用层协议的解析，Processor 解析后生成 Tomcat Request 对象，然后调用 Adapter 的`service()`方法，方法内部通过下面代码将 Request 请求传递到容器中。

```java
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
```

一个总的 Connector 功能：

![1](https://drun1baby.top/2022/08/19/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-01-%E5%9F%BA%E7%A1%80%E5%86%85%E5%AE%B9%E5%AD%A6%E4%B9%A0/TomcatConnector.png)

#### Container

Container（也叫 Catalina）用于处理 Connector 发过来的 Servlet 连接请求，它是容器的父接口，所有子容器都必须实现这个接口，Container 容器的设计用的是典型的责任链的设计模式，它有四个子容器组件构成，即 Engine、Host、Context、Wrapper，这四个组件不是平行的，而是父子关系，Engine 包含 Host，Host 包含 Context，Context 包含 Wrapper。

- Engine: 最顶层容器组件，可以包含多个 Host。实现类为 `org.apache.catalina.core.StandardEngine`
- Host: 代表一个虚拟主机，每个虚拟主机和某个域名 Domain Name 相匹配，可以包含多个 Context。实现类为 `org.apache.catalina.core.StandardHost`
- Context: 一个 Context 对应于一个 Web 应用，可以包含多个 Wrapper。实现类为 `org.apache.catalina.core.StandardContext`
- Wrapper: 一个 Wrapper 对应一个 Servlet。负责管理 Servlet ，包括 Servlet 的装载、初始化、执行以及资源回收。实现类为 `org.apache.catalina.core.StandardWrapper`

通常一个 Servlet class 对应一个 Wrapper，如果有多个 Servlet 就可以定义多个 Wrapper，如果有多个 Wrapper 就要定义一个更高的 Container。

比如：

![1](https://drun1baby.top/2022/08/19/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-01-%E5%9F%BA%E7%A1%80%E5%86%85%E5%AE%B9%E5%AD%A6%E4%B9%A0/TomcatContainer.png)

每个 Context 都有唯一的 path。这里的 path 不是指 Servlet 绑定的 WebServlet 地址，而是指独立的一个 Web 应用地址。就好比 Tomat 默认的 / 地址和 /manager 地址就是两个不同的 web 应用，所以对应两个不同的 Context。要添加 Context 需要在 server.xml 中配置 docbase。

如下图所示， 在一个 web 应用中创建了 2 个 servlet 服务，WebServlet 地址分别是 /Demo1 和 /Demo2 。 因为它们属于同一个 Web 应用所以 Context 一样，但访问地址不一样所以 Wrapper 不一样。 /manager 访问的 Web 应用是 Tomcat 默认的管理页面，是另外一个独立的 web 应用， 所以 Context 与前两个不一样。

![1](https://drun1baby.top/2022/08/19/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-01-%E5%9F%BA%E7%A1%80%E5%86%85%E5%AE%B9%E5%AD%A6%E4%B9%A0/TomcatContext.png)

## Tomcat 的类加载机制

由于 Tomcat 中有多个 WebApp 同时要确保之间相互隔离，所以 Tomcat 的类加载机制也不是传统的双亲委派机制。

Tomcat 自定义的类加载器 WebAppClassLoader 为了确保隔离多个 WebApp 之间相互隔离，所以打破了双亲委派机制。每个 WebApp 用独有的 ClassLoader 实例来优先处理加载。它首先尝试自己加载某个类，如果找不到再交给父类加载器，其目的是优先加载 Web 应用自己定义的类。

同时为了防止 Web 应用自己的类覆盖 JRE 的核心类，在本地 Web 应用目录下查找之前，先使用 ExtClassLoader（使用双亲委派机制）去加载，这样既打破了双亲委派，同时能安全加载类。

































