---
title: JDK动态代理
date: 2026-05-01
tags: 
  - java安全
  - JDK动态代理
categories:
  - web安全
  - java安全
excerpt: JDK动态代理初探
---

# JDK动态代理

## 代理模式

核心定义：为目标对象提供一个代理对象，并由代理对象控制对目标对象的方法。代理对象可以在目标对象执行的前后，进行额外的增强处理，目标对象只需要专注自己的核心业务逻辑即可。

## 静态代理

### 简单理解

角色分析：

- 抽象角色：一般使用接口或抽象类来解决
- 真实角色：被代理的角色
- 代理角色：代理真实角色，代理真实角色后会做一些附属操作
- 客户：访问代理的人

以租客找中介向房东租房为例：

Rent.java

```java
package JDKproxy.staticProxy;

public interface Rent {
    public void rent();
}
```

租房这件事是需要中介和房东完成的，是一个抽象角色，所以需要用接口实现。

Host.java

```java
package JDKproxy.staticProxy;

public class Host implements Rent{
    @Override
    public void rent() {
        System.out.println("房东出租房子");
    }
}
```

这是房东类，需要实现 Rent 接口。

RentProxy.java

```java
package JDKproxy.staticProxy;

public class RentProxy implements Rent{

    private Host host;

    public RentProxy(Host host) {
        this.host = host;
    }

    public RentProxy() {
    }

    @Override
    public void rent() {
        host.rent();
        seeHouse();
        fare();
        contract();
    }

    public void seeHouse(){
        System.out.println("中介看房");
    }

    public void fare(){
        System.out.println("收中介费");
    }

    public void contract(){
        System.out.println("签合同");
    }
}
```

这是中介类，也就是代理，中介代表房东出租房子，同时也会提供一些额外的服务。

Client.java

```java
package JDKproxy.staticProxy;

public class Client {
    public static void main(String[] args) {
        Host host = new Host();
        RentProxy rentProxy = new RentProxy(host);
        rentProxy.rent();
    }
}
```

这是一个启动类，也就是上面讲的客户，客户的需求很简单，就是找到中介（代理），然后租房。途中还会有一些其他的操作，这些就都是由中介（代理）完成的。

静态代理的优点：

1. 逻辑清晰，容易理解，不需要反射，性能较好。
2. 让真实角色更加纯粹，不需要去关注公共的事情。
3. 公共的业务由代理完成，实现分工。
4. 公共业务拓展时更加集中且方便。

缺点：

- 一个真实类对应一个代理，一旦类多了，代码量直接翻倍，效率太低。

我们需要静态代理的优点同时要减少代码量，就出现了动态代理。

## 动态代理

静态代理的代理类是我们手写的，而动态代理的代理类是在运行时由 JVM 动态生成的，无需手动创建`.java`文件。它是 JDK 原生支持的，前提是目标对象必须实现接口。

动态代理的核心是一个接口和一个代理类：

- `InvocationHandler` 接口：额外的逻辑写在这里，所有代理方法的调用，最终都会进入`invoke`方法中。
- `Proxy`类：用它来动态生成并加载代理类。

下面先给一个样例，然后根据样例来学习动态代理。

### 样例

接口：UserService.java

```java
package JDKproxy.DynamicProxy;

public interface UserService {
    public void add();
    public void delete();
    public void update();
    public void query();
}
```

被代理的类：UserServiceImpl.java

```java
package JDKproxy.DynamicProxy;

public class UserServiceImpl implements UserService{
    @Override
    public void add() {
        System.out.println("添加");
    }

    @Override
    public void delete() {
        System.out.println("删除");
    }

    @Override
    public void update() {
        System.out.println("更新");
    }

    @Override
    public void query() {
        System.out.println("查询");
    }
}
```

调用处理器：UserProxyInvocationHandler.java

```java
package JDKproxy.DynamicProxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class UserProxyInvocationHandler implements InvocationHandler {
    private UserServiceImpl userService;

    public UserProxyInvocationHandler(UserServiceImpl userService) {
        this.userService = userService;
    }

    @Override         //这里的参数都是 JVM 自动获取的，不需要手动设置
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        method.invoke(userService,args);
        return null;
    }
}
```

启动类：ProxyTest.java

```java
package JDKproxy.DynamicProxy;

import java.lang.reflect.Proxy;

public class ProxyTest {
    public static void main(String[] args) {
		//创建一个真实对象
        UserServiceImpl userService = new UserServiceImpl();
		//创建调用处理器，以后所有对代理方法的调用都会经过这个处理器的invoke方法
        UserProxyInvocationHandler userProxyInvocationHandler = new UserProxyInvocationHandler(userService);
        //动态生成代理对象
        //JVM 底层会做这些事:
        //a) 在内存中动态拼出一个 .class 字节码,这个类实现了 UserService 接口
        //b) 这个类长这样(伪代码):
        //           class $Proxy0 implements UserService {
        //               InvocationHandler h;
        //               public final void add() {
        //                   h.invoke(this, m3, null);  // m3 就是 add 的 Method 对象
        //               }
        //               public final void delete() {
        //                   h.invoke(this, m4, null);
        //               }
        //               ... update() / query() 同理
        //           }
        //        c) 用 classLoader 把这个类加载到 JVM
        //        d) 创建该类的实例,把 userProxyInvocationHandler 赋给它的 h 字段
        JDKproxy.DynamicProxy.UserService userproxy = (UserService) Proxy.newProxyInstance(userService.getClass().getClassLoader(), userService.getClass().getInterfaces(), userProxyInvocationHandler);
        // 步骤4: 调用代理对象的 add() 方法
        //
        //        userproxy 是 $Proxy0 的实例,调用 add() 时:
        //        userproxy.add()
        //            → $Proxy0.add() 被 JVM 调用
        //            → 内部执行 this.h.invoke(this, method, args)
        //            → 进入 UserProxyInvocationHandler.invoke()
        //            → invoke 里用反射: method.invoke(真实对象, args)
        //            → 真实对象 UserServiceImpl.add() 执行
        //            → 打印 "添加"        
        userproxy.add();
    }
}
```

注：动态代理代理的是接口。代理类是动态生成的，不是静态确定的。

我们生成的代理对象要强转成接口类型，一方面是为了能够调用接口声明的方法，另一方面是因为 java 是单继承机制，而我们生成的代理对象已经继承了 Proxy 类，所以在需要代理对象和真实对象的类型相同的要求下，使用接口类型是必须的。

## 总结

这篇博客主要是对之前的 JDK 动态代理进行一个整理，顺便理一下思路，目前对于动态代理的内部运行逻辑有了大致的认识，但是对于底层是怎么实现的并没有进行研究，不过目前还是 java 安全基础阶段，或许以后会去研究 Java 底层的一些原理。



















