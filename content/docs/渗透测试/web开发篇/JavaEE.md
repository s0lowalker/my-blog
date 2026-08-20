---
title: JavaEE
date: 2026-08-19
---

# JavaEE

## Servlet 技术

### 整体理解

Servlet 是运行在 Web 服务器或应用服务器上的程序，它是作为来自 Web 浏览器或其他 HTTP 客户端的请求和 HTTP 服务器上的数据库或应用程序之间的中间层。使用 Servlet 可以收集来自网页表单的用户输入，呈现来自数据库或者其他源的记录，还可以动态创建网页。

在纯 Servlet 项目中一般会用到 Tomcat。这里我们和 PHP 的经典架构 LAMP 进行一个类比。

在 LAMP 中，当 Apache 接收到请求后会自己处理静态内容，如 HTML、CSS 等，然后把这些内容直接返回给客户端，而动态请求交给后续的 PHP 处理，比如查询数据库等。

到 Java Web 架构中，也可以使用 Apache/Nginx 来处理静态资源，然后动态请求就会发到 Tomcat 处理，再由 Tomcat 交给 Servlet 处理并生成对应内容并返回客户端。当然也可以不使用 Apache/Nginx 这类应用，因为 Tomcat 本身具备一个简易的 Web Server，也可以处理 HTTP 请求，然后交给内部的 Servlet 处理请求。

### Tomcat 与 Servlet

Tomcat 是 Servlet 容器。Servlet 类写好之后是没有 main 方法的，无法自行启动。我们必须把 Servlet 放到 Tomcat 中 Tomcat 才能创建 Servlet 实例，并在合适的时机调用它的方法。同时 Tomcat 管理了 Servlet 的完整生命周期。当启动或第一次请求时，Tomcat 调用`init()`方法，让 Servlet 初始化；每一次处理请求时，Tomcat 开线程，调用`service()`方法（进而调用我们的`doGet`/`doPost`方法）；服务器关闭或应用卸载时，Tomcat 调用`destroy()`方法，让 Servlet 释放资源。

同时，Tomcat 可以接收原始的 TCP 字节流并解析为 HTTP 请求，然后把各种请求的参数封装到`HttpServletRequest`对象中，我们可以直接调用`request.getParameter("xxx")`来获取参数。在处理 HTTP 响应时我们把结果写到`HttpServletResponse`对象中，Tomcat 会自动把它组装成合法的 HTTP 响应报文发送到客户端。当多个用户同时访问时，Tomcat 会自动分配多个线程来调用 Servlet，并保证每个请求之间互不干扰。而向长连接管理、SSL/TLS 加密、会话管理等都是由 Tomcat 提供的。

### 样例代码

```java
package com.example.servlet;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;
//一个简单的servlet样例
public class IndexServlet extends HttpServlet {
    @Override
    public void init() throws ServletException {
        System.out.println("init servlet");
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String name = req.getParameter("name");
        resp.setContentType("text/html");
        PrintWriter out = resp.getWriter();
        out.println(name);
        System.out.println("doGet servlet");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("doPost servlet");
    }

    @Override
    public void destroy() {
        System.out.println("destroy servlet");
    }
}
```

同时需要给这个文件配置路由，如果不配置路由则无法调用这个文件的相关方法。

第一种配置路由的方法是在 web.xml 种添加相关配置，比如添加：

```xml
<servlet>
	<servlet-name>index</servlet-name>
	<servlet-class>com.example.servlet.IndexServlet</servlet-class>
</servlet>

<servlet-mapping>
	<servlet-name>index</servlet-name>
	<url-pattern>/index</url-pattern>
</servlet-mapping>
```

第二种方法更加简单，直接在这个类的上面加上一行注解即可：

```java
@WebServlet(name="index",value="/index")
```

## Filter 过滤器

![屏幕截图 2026-08-05 214416](D:/hugoblog/myblog/static/images/screenshots/屏幕截图 2026-08-05 214416.png)

通过这一张图我们可以清楚地看到客户端访问时的数据走向。

Filter被称为过滤器，过滤器实际上就是对Web资源进行拦截，做一些处理后再交给下一个过滤器或Servlet处理，通常都是用来拦截request进行处理的，也可以对返回的 response进行拦截处理。开发人员利用filter技术，可以实现对所有Web资源的管理，例如实现权限访问控制、过滤敏感词汇、压缩响应信息等一些高级功能。

### 样例代码

```java
package com.example.servlet.Filter;

import javax.servlet.*;
import java.io.IOException;

public class XssFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("XssFilter init");
    }

    @Override
    public void destroy() {
        System.out.println("XssFilter destroy");
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("XssFilter doFilter");
        chain.doFilter(request,response);   //数据放行
    }
}
```

然后在 web.xml 里配置一下相关路由：

```xml
<filter>
	<filter-name>xss</filter-name>
	<filter-class>com.example.servlet.Filter.XssFilter</filter-class>
</filter>
    
<filter-mapping>
	<filter-name>xss</filter-name>
	<url-pattern>/index</url-pattern>
</filter-mapping>
```

或者配置注解：

```java
@WebFilter(filterName = "xss",value="/index")
```

然后运行代码，会发现我们还没有访问`/index`这个路由时，已经输出了`XssFilter init`。然后我们访问`/index`路由，先输出`init servlet`，再输出`XssFilter doFilter`。然后停止程序运行，先输出`destroy servlet`，再输出`XssFilter destroy`。

一个简单的 XSS 过滤：

```java
package com.example.servlet.Filter;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

public class XssFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("XssFilter init");
    }

    @Override
    public void destroy() {
        System.out.println("XssFilter destroy");
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain) throws IOException, ServletException {
        System.out.println("XssFilter doFilter");
        HttpServletRequest request = (HttpServletRequest) servletRequest;

        String name = request.getParameter("name");
        if(!name.contains("script")){
            chain.doFilter(servletRequest,servletResponse);
        }else {
            System.out.println("XSS attack");
        }
    }
}
```

过滤器就是数据在到达核心程序之前进行一次过滤。后续内存马技术会涉及到过滤器。

## Listener 监听器

监听器是 Servlet 规范中定义的一种特殊类：

- 用来监听 ServletContext、HttpSession 和 ServletRequest 等域对象的创建和销毁事件
- 用来监听域对象的属性发生修改的事件
- 可以在事件发生前后做一些必要的处理

Servlet规范中定义了9个监听器接口，可以用来监听ServletContext、HttpSession 和 ServletRequest 对象的生命周期 和属性变化事件。

监听器按照监听的事件可以分成3类：

- 监听对象创建和销毁的监听器
- 监听对象属性变更的监听器
- 监听 HttpSession 中的对象状态改变的监听器

关于监听器的更多了解参考[Web应用监听器实战：在线人数统计与事件处理-CSDN博客](https://blog.csdn.net/qq_52797170/article/details/124023760)

### 样例代码

下面是一个简单的 Session 监听器：

```java
package com.example.servlet.Listen;

import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpSessionEvent;
import javax.servlet.http.HttpSessionListener;

@WebListener("/admin")
public class SessionListen implements HttpSessionListener {
    @Override
    public void sessionCreated(HttpSessionEvent se) {
        System.out.println("Listen Session created");
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        System.out.println("Listen Session destroyed");
    }
}
```

```java
package com.example.servlet.Servlet;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@WebServlet(name="admin",value="/admin")
public class AdminServlet extends HttpServlet {
    @Override
    public void init() throws ServletException {
        System.out.println("AdminServlet init");
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("AdminServlet doGet");
        req.getSession().invalidate();
    }

    @Override
    public void destroy() {

    }
}
```

## ORM 框架

Java Web 中的 ORM 框架就是一个把 Java 对象和数据库表记录之间自动相互转换的工具。

### 原生 JDBC

```java
package com.example.jdbc.servlet;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.sql.*;

@WebServlet(name="jdbc",value="/sql")
public class JdbcServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String id=req.getParameter("id");
        String sql="select * from users where user_id="+id;
        String url="jdbc:mysql://localhost:3306/dvwa?useUnicode=true&characterEncoding=utf-8";
        try {
            //加载驱动
            Class.forName("com.mysql.jdbc.Driver");
            Connection connection = DriverManager.getConnection(url, "root", "123456");    //连接数据库
            //创建SQL语句执行器
            Statement statement = connection.createStatement(); 
            //执行查询语句
            ResultSet resultSet = statement.executeQuery(sql);
            while (resultSet.next()){
                //按照列名查找数据并转换为字符串类型，输出到浏览器上
                resp.getWriter().println(resultSet.getString("user_id"));
                resp.getWriter().println(resultSet.getString("first_name"));
                resp.getWriter().println(resultSet.getString("last_name"));
                resp.getWriter().println(resultSet.getString("user"));
            }
        } catch (ClassNotFoundException | SQLException e) {
            throw new RuntimeException(e);
        }
    }
}
```

以上是一个简单的原生 JDBC 代码，存在 SQL 注入漏洞。这么写既危险又繁琐，所以出现了其他 ORM 框架，既能简化代码，又能提高安全性。

#### JDBC 中的 SQL 预编译

为了提高原生 JDBC 的安全性、防止存在 SQL 注入，JDBC 中也有预编译机制。

```java
package com.example.jdbc.servlet;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.sql.*;

@WebServlet(name="jdbc",value="/sql")
public class JdbcServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String id=req.getParameter("id");

        //不安全写法
        //String sql="select * from users where user_id="+id;

        //预编译
        String sql="select * from users where user_id=?";

        String url="jdbc:mysql://localhost:3306/dvwa?useUnicode=true&characterEncoding=utf-8";
        try {
            Class.forName("com.mysql.jdbc.Driver");
            Connection connection = DriverManager.getConnection(url, "root", "123456");

            //Statement statement = connection.createStatement();
            //ResultSet resultSet = statement.executeQuery(sql);

            //预编译
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1,id);
            ResultSet resultSet = preparedStatement.executeQuery();

            while (resultSet.next()){
                resp.getWriter().println(resultSet.getString("user_id"));
                resp.getWriter().println(resultSet.getString("first_name"));
                resp.getWriter().println(resultSet.getString("last_name"));
                resp.getWriter().println(resultSet.getString("user"));
            }
        } catch (ClassNotFoundException | SQLException e) {
            throw new RuntimeException(e);
        }
    }
}
```

这就是原生 JDBC 中简单的 SQL 预编译代码，可以防御 SQL 注入漏洞。

### MyBatis

MyBatis 的配置相对复杂，这里也不记录如何配置了，直接放样例代码。

mybatis-config.xml：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/dvwa?serverTimezone=UTC"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="UserMapper.xml"/>
    </mappers>
</configuration>
```

这个配置主要负责连接数据库。

UserMapper.xml：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mybatis.mapper.UserMapper">
    <select id="findUserByName" parameterType="String" resultType="com.example.mybatis.model.User">
        select * from users where user = '${name}'
    </select>
</mapper>
```

这个配置就是负责对数据库进行查询，同时 SQL 注入点也在这里。`${name}`是不安全的写法，会导致 SQL 注入，而`#{name}`就是预编译写法，更加安全。

UserMapper.java：

```java
package com.example.mybatis.mapper;

import com.example.mybatis.model.User;
import org.apache.ibatis.annotations.Param;

import java.util.List;

public interface UserMapper {
    List<User> findUserByName(@Param("name") String name);
}
```

这个接口来接收参数，并且会填充到 SQL 语句中。

```java
package com.example.mybatis.model;

public class User {
    private int user_id;
    private String user;
    private String password;

    @Override
    public String toString() {
        return "User{" +
                "user_id=" + user_id +
                ", user='" + user + '\'' +
                ", password='" + password + '\'' +
                '}';
    }

    public int getUser_id() {
        return user_id;
    }

    public void setUser_id(int user_id) {
        this.user_id = user_id;
    }

    public String getUser() {
        return user;
    }

    public void setUser(String user) {
        this.user = user;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

这个类接收查询到的数据。

```java
package com.example.mybatis.servlet;

import com.example.mybatis.mapper.UserMapper;
import com.example.mybatis.model.User;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.InputStream;
import java.util.List;

@WebServlet(name="user",value="/user")
public class UserServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String name = req.getParameter("name");

        if(name==null||name.isEmpty()){
            resp.getWriter().write("username can't be empty");
            return;
        }

        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        // 获取 SqlSession
        try (SqlSession session = sqlSessionFactory.openSession()) {
            // 获取 Mapper 接口
            UserMapper mapper = session.getMapper(UserMapper.class);

        // 执行查询(注意:${name} 为拼接注入点,返回 List)
            List<User> users = mapper.findUserByName(name);

            // 输出结果
            if (users != null && !users.isEmpty()) {
                for (User user : users) {
                    resp.getWriter().println("ID: " + user.getUser_id());
                    resp.getWriter().println("username: " + user.getUser());
                    resp.getWriter().println("password: " + user.getPassword());
                    resp.getWriter().println("--------------------");
                }
            } else {
                resp.getWriter().write("111111no");
            }
        } catch (Exception e) {
            // 渗透测试场景:把报错信息回显出来,方便报错注入/看列名
            e.printStackTrace();
            resp.getWriter().write("failed: " + e.getMessage());
        }
    }
}
```

这段代码负责处理用户输入和数据库查询。

### Hibernate

样例代码：

先创建一个 hibernate.cfg.xml：

```xml
<?xml version='1.0' encoding='utf-8'?>
<!DOCTYPE hibernate-configuration PUBLIC
        "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>
        <!--  数据库连接配置  -->
        <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
        <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/phpstudy?useUnicode=true&amp;characterEncoding=UTF-8&amp;serverTimezone=Asia/Shanghai</property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password">123456</property>
        <!--  数据库方言  -->
        <property name="hibernate.dialect">org.hibernate.dialect.MySQL8Dialect</property>
        <!--  显示 SQL 语句  -->
        <property name="hibernate.show_sql">true</property>
        <!--  自动更新数据库表结构  -->
        <property name="hibernate.hbm2ddl.auto">update</property>
        <!--  映射实体类  -->
        <mapping class="com.example.entity.User"/>
    </session-factory>
</hibernate-configuration>
```

User.java 映射实体类：

```java
package com.example.entity;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;

@Entity
@Table(name="users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int id;
    private String username;
    private String email;

    public User() {}

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    @Override
    public String toString() {
        return "User{id=" + id + ", username='" + username + "', email='" + email + "'}";
    }
}
```

工具类：

```java
package com.example.util;

import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;

public class HibernateUtil {
    private static final SessionFactory sessionFactory;

    static {
        try {
            Configuration configuration = new Configuration().configure();
            sessionFactory = configuration.buildSessionFactory();
        } catch (Throwable ex) {
            throw new ExceptionInInitializerError(ex);
        }
    }

    public static SessionFactory getSessionFactory() {
        return sessionFactory;
    }
}
```

Servlet 类部分：

```java
package com.example;

import com.example.entity.User;
import com.example.util.HibernateUtil;
import org.hibernate.Session;
import org.hibernate.query.Query;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;
import java.util.List;

@WebServlet("/user")
public class UserQueryServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        response.setContentType("text/html;charset=UTF-8");
        PrintWriter out = response.getWriter();

        // 获取参数（这里假设根据用户名查询）
        String username = request.getParameter("username");

        // 打开 Hibernate Session
        Session session = HibernateUtil.getSessionFactory().openSession();
        try {
            // 修改后的 HQL 语句，直接拼接字符串
            String hql = "FROM User WHERE username = '" + username + "'";

            //String hql = "FROM User WHERE username=:username";
            // 创建查询对象
            Query<User> query = session.createQuery(hql, User.class);
            // 执行查询
            List<User> users = query.getResultList();

            // 输出查询结果
            out.println("<html><body>");
            if (users.isEmpty()) {
                out.println("<p>未找到匹配的用户。</p>");
            } else {
                for (User user : users) {
                    out.println("<p>" + user + "</p>");
                }
            }
            out.println("</body></html>");
        } catch (Exception e) {
            e.printStackTrace();
            out.println("<html><body><p>查询出错，请稍后重试。</p></body></html>");
        } finally {
            // 关闭 Session
            session.close();
        }
    }
}
```

这里的代码是存在 SQL 注入的。在 Hibernate 中用预编译来写就是`String hql = "FROM User WHERE username=:username"`。

## Fastjson

### 简介

Fastjson 是阿里巴巴的开源库，用于对 JSON 格式的数据进行解析和打包。其实简单的来说就是处理 json 格式的数据。例如将 JSON 转换成一个类，或者是将一个类转换成一段 json 数据。Fastjson 是一个 Java 库，提供了 Java 对象与 JSON 相互转换。

### 序列化与反序列化

1. 序列化方法

`JSON.toJsonString()`，返回字符串

`JSON.toJSONBytes()`，返回 byte 数组

2. 反序列化方法

`JSON.parseObject()`，返回`JsonObject`

`JSON.parse()`，返回`Object`

`JSON.parseArray()`，返回`JSONArray`

将 JSON 对象转换为 java 对象：`JSON.toJavaObject()`

将 JSON 对象写入 write 流：`JSON.writeJSONString()`

3. 常用

`JSON.toJSONString()`、`JSON.parse()`、`JSON.parseObject()`

### 代码demo

#### 序列化实现

首先导入 Fastjson 的依赖，这里使用 1.2.24 版本的：

```xml
<dependency>  
 <groupId>com.alibaba</groupId>  
 <artifactId>fastjson</artifactId>  
 <version>1.2.24</version>  
</dependency>
```

先定义一个 Student 类：

```java
public class Student {
    private String name;
    private int age;

    public Student() {
        System.out.println("构造函数");
    }

    public String getName() {
        System.out.println("getName");
        return name;
    }

    public void setName(String name) {
        System.out.println("setName");
        this.name = name;
    }

    public int getAge() {
        System.out.println("getAge");
        return age;
    }

    public void setAge(int age) {
        System.out.println("setAge");
        this.age = age;
    }
}
```

然后用`JSON.toJSONString()`来序列化：

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.serializer.SerializerFeature;

public class StudentSerialize {
    public static void main(String[] args) {
        Student student = new Student();
        student.setName("solowalker");

        String jsonString = JSON.toJSONString(student, SerializerFeature.WriteClassName);
        System.out.println(jsonString);
    }
}
```

输出：

```text
构造函数
setName
getAge
getName
{"@type":"com.example.Student","age":0,"name":"solowalker"}
```

很显然`String jsonString = JSON.toJSONString(student, SerializerFeature.WriteClassName);`这一句是非常关键的。

我们关注一下这里的参数：

第一个参数是 student，就是我们要序列化的对象。第二个参数是`SerializerFeature.WriteClassName`，是`JSON.toJSONString()`中的一个设置属性值，设置之后在序列化时会多写入一个`@type`，即会写上被序列化的类名，type 可以指定反序列化的类，并且反序列化时调用其`getter`/`setter`/`is`方法。

- Fastjson 接受的 JSON 可以通过@type字段来指定该JSON应当还原成何种类型的对象，在反序列化的时候方便操作。

```text
// 设置了SerializerFeature.WriteClassName
构造函数
setName
setAge
getAge
getName
{"@type":"com.example.Student","age":6,"name":"John"}
 
// 未设置SerializerFeature.WriteClassName
构造函数
setName
setAge
getAge
getName
{"age":6,"name":"John"}
```

#### 反序列化实现

调用`JSON.parseObject()`：

```java
package com.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;

public class StudentUnserialize {
    public static void main(String[] args) {
        String jsonString="{\"@type\":\"com.example.Student\",\"age\":0,\"name\":\"solowalker\"}";
        Student student = JSON.parseObject(jsonString, Student.class, Feature.SupportNonPublicField);
        System.out.println(student);
        System.out.println(student.getClass().getName());
    }
}
```

输出：

```text
构造函数
setAge
setName
com.example.Student@5b37e0d2
com.example.Student
```

总结一下反序列化时 getter 和 setter 的调用情况：

1. 当序列化时有`SerializerFeature.WriteClassName`参数，即 json 中有`@type`时，反序列化时`parse`就会调用`setter`，`parseObject`就是调用`getter`和`setter`
2. 当反序列化时指定`Class`，`parseObject(text, Class)`主要会调用`setter`，特定情况下会调用`getter`

## JNDI

### 简介

JNDI 全称 Java Naming and Directory Interflace（Java 命名和目录接口），也就是一个名字对应一个 Java 对象，一个字符串对应一个对象。

JNDI 是一组应用程序接口，为开发人员查找和访问各种资源提供了统一的通用接口，可以用来定义用户、网络、机器、对象和服务等各种资源。JNDI 支持的服务主要有：DNS、LDAP、CORBA、RMI 等。

关于 JNDI 的攻击方式后续会在 Java 安全专题里更新。

## Log4j

### 简介

Log4j 是 Java 编程中最流行的一个**日志记录工具**。简单来说就是开发人员在代码里“埋点”，让程序在运行过程中把各种状态、错误、信息打印出来（比如输出到控制台或文件），方便开发人员排查问题和监控系统。

在大型企业级应用中，日志系统是必备的。Log4j 因为性能好、功能强、配置灵活，几乎成了 Java 日志的事实标准，被全球数以百万计的应用使用。

### 样例代码

```java
package com.example.log4jdemo;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class Log4jTest {
    private static final Logger log = LogManager.getLogger(Log4jTest.class);

    public static void main(String[] args) {
        log.error("hello");

        String code="${java:os}";
        log.error("{}",code);
    }
}
```

这段代码简单利用了 Log4j 的漏洞打印了系统的基本信息。

## XStream

XStream 是一个 Java 库，核心功能是把 Java 对象转换为 XML、把 XML 转换成 Java 对象，就是序列化和反序列化的过程。

### 样例代码

```java
package com.example.xstreamdemo;

import java.io.ObjectInputStream;
import java.io.Serializable;

public class Car implements Serializable {
    private String name;
    private int price;

    public Car(String name, int price) {
        this.name = name;
        this.price = price;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getPrice() {
        return price;
    }

    public void setPrice(int price) {
        this.price = price;
    }

    private void readObject(ObjectInputStream s) throws Exception{
        s.defaultReadObject();
        System.out.println("print car");
        Runtime.getRuntime().exec("calc");
    }
}
```

```java
package com.example.xstreamdemo;

import com.thoughtworks.xstream.XStream;

public class XStreamTest {
    public static void main(String[] args) {
        Car ferrari = new Car("Ferrari", 4000000);
        XStream xStream = new XStream();
        String xml = xStream.toXML(ferrari);
        System.out.println(xml);
    }
}
```

输出：

```text
<com.example.xstreamdemo.Car serialization="custom">
  <com.example.xstreamdemo.Car>
    <default>
      <price>4000000</price>
      <name>Ferrari</name>
    </default>
  </com.example.xstreamdemo.Car>
</com.example.xstreamdemo.Car>
```

反序列化：

```java
package com.example.xstreamdemo;

import com.thoughtworks.xstream.XStream;

public class XStreamSer {
    public static void main(String[] args) {
        XStream xStream = new XStream();
        String payload="<com.example.xstreamdemo.Car serialization=\"custom\">\n" +
                "  <com.example.xstreamdemo.Car>\n" +
                "    <default>\n" +
                "      <price>4000000</price>\n" +
                "      <name>Ferrari</name>\n" +
                "    </default>\n" +
                "  </com.example.xstreamdemo.Car>\n" +
                "</com.example.xstreamdemo.Car>";
        xStream.fromXML(payload);
    }
}
```

运行之后会弹出计算器，说明 XStream 在反序列化时会调用`readObject()`方法。

## shiro

Apache Shiro 是一个强大且易用的 Java 安全框架，它的目标是让应用程序的安全控制变的简单直观。它关注的是应用安全的四大基石，即认证、授权、会话管理和加密。

Shiro 可以轻松实现以下功能：

- **认证 (Authentication)**：也就是“登录”，验证用户的身份。
- **授权 (Authorization)**：也就是“访问控制”，判断用户是否有权限做某件事。
- **会话管理 (Session Management)**：即使在非 Web 环境（如命令行程序）中，也能管理用户会话。
- **加密 (Cryptography)**：提供简单的 API 来使用加密算法保护数据。
- **其他功能**：它还支持“记住我”、缓存、多线程并发等特性。

## SpringBoot

SpringBoot 是 Java 生态中最流行的应用开发框架（基于 Spring 框架），核心目标是简化 Spring 应用的初始搭建和开发过程。

在 Spring Boot 出现之前，用 Spring 开发一个 Web 项目，你需要：

- 配置大量的 XML 文件
- 手动管理各种 Jar 包的版本依赖
- 部署到外置的 Tomcat 服务器
- 配置数据源、事务管理器等基础设施

这些工作繁琐且容易出错。Spring Boot 的核心理念是**"约定优于配置"**——它给你一套默认的配置，只要你遵循约定，几乎不需要额外配置就能直接运行。

### 模板引擎

Java 中用到的模板引擎主要是 Thymeleaf、Freemarker、Velocity。Java 的模板引擎同样存在 SSTI 漏洞。

关于这些模板引擎的漏洞后续专题更新。

### Actuator

Spring Boot Actuator 是 Spring Boot 框架中一个非常重要的模块，它为应用提供了一系列**生产就绪的功能**，比如监控、管理和查看应用内部状态 。你可以把它看作是应用运行时的“体检中心”和“控制面板”，让你能轻松了解应用的健康状况、性能指标和配置信息 。

简单来说，Actuator 通过暴露一系列 **HTTP 端点**（或 JMX MBean）来提供功能。这些端点就像一个个“接口”，通过访问它们就能获取或操作应用的内部信息 。

| 端点分类           | 常用端点示例                           | 功能介绍                                                     |
| :----------------- | :------------------------------------- | :----------------------------------------------------------- |
| **应用健康与状态** | `/health`, `/info`                     | 检查应用是否存活、就绪，以及获取应用的自定义信息，是 Kubernetes 探针的基础 。 |
| **性能与度量指标** | `/metrics`, `/threaddump`, `/heapdump` | 获取 JVM 内存使用、CPU 负载、线程状态、堆转储等关键指标，用于性能分析 。 |
| **应用配置与环境** | `/env`, `/configprops`, `/beans`       | 查看应用所有的环境变量、配置属性以及 Spring 容器中的所有 Bean，方便排查配置问题 。 |
| **操作与控制**     | `/shutdown`, `/loggers`                | 允许远程关闭应用（需谨慎开启），或在运行时动态调整日志级别 。 |

### Swagger

Swagger 是当下比较流行的实时接口文档生成工具。接口文档是当前前后端分离项目中必不可少的工具，在前后端开发之前，后端要先出接口文档，前端根据接口文档来进行项目的开发，双方开发结束后再进行联调测试。

在前后端分离的开发模式中，它能解决以下痛点：

- **消除信息孤岛**：后端修改了接口（如改变了参数名），Swagger文档会**实时刷新**，前端能立刻看到最新定义，无需口头沟通或等待文档更新。
- **降低测试成本**：Swagger-UI页面提供“**Try it out**”按钮，开发者无需使用Postman，在浏览器中就能直接填充参数、发送请求、查看JSON返回结果。
- **规范接口设计**：通过强类型约束和注解，倒逼开发者设计出更清晰的入参和出参结构。

### JWT 令牌

JWT（JSON Web Token）是由服务端用加密算法对信息签名来保证其完整性和不可伪造；Token 中可以包含所有必要信息，这样服务端就不需要保存任何关于用户或会话的信息；JWT 用于身份认证、会话维持等，由三部分组成：header、payload 与 signature。

JWT 的安全问题一般就两点：

1. 目标使用的空加密
2. 爆破密钥，但是复杂的密钥几乎无法爆破

### Spring Security

Spring Security 是 Spring 生态中为应用提供**身份认证（你是谁）**和**授权（你能干什么）**能力的基石。

Spring Security 的核心本质，就是一套**过滤器链（Filter Chain）**。一个请求进来，需要经过层层过滤器的检查。其中最重要的两个核心概念是：

- **认证（Authentication）**：验证**你是谁**。常见方式是用户名+密码、手机号验证码、JWT（JSON Web Token）令牌等。
- **授权（Authorization）**：验证**你能做什么**。即认证通过后，判断你是否有权限访问某个资源或执行某个操作（比如普通用户不能调用管理员的删除接口）。

这两个概念在代码中分别对应两个核心接口：

- `Authentication`：代表当前用户的身份凭证。
- `GrantedAuthority`：代表用户拥有的权限（通常表现为角色或权限字符串）。

## SnakeYaml

在Java生态里，SnakeYAML 是最主流、最成熟的 **YAML 解析库**。它几乎成了“在Java里处理YAML”的代名词，尤其是在 Spring Boot 这类广泛使用 YAML 做配置文件的框架中，它的身影随处可见。

简单来说，SnakeYAML 是一个 Java 库，专门用来在 YAML 格式和 Java 对象之间进行转换。YAML 是一种对人极其友好的数据序列化格式，常被用来写配置文件，它比 XML 和 Properties 文件都更清晰易读。

序列化时会调用对象的构造方法、`getter`和`setter`；反序列化时会调用构造方法和`setter`。

SnakeYaml 有一个 SPI 机制，解决了 JNDI 注入无法绕过的情况，存在高危利用。

























