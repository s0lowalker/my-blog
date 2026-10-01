---
title: Spring开发
date: 2026-10-01
---

# Spring 开发

## 简介

### 关于 Spring

Spring 理念：使现有技术更加容易使用。Spring 本身就是一个大杂烩，整合现有的框架技术。

- SSH：Struts2 + Spring + Hibernate
- SSM：SpringMVC + Spring + Mybatis

Spring 是一个轻量级控制反转（IOC）和面向切面（AOP）的容器框架。

**优点：**

- Spring 是一个开源免费的框架
- Spring 是一个轻量级的、非入侵式的框架
- 控制反转（IOC）、面向切面编程（AOP）
- 支持事务的处理，对框架的整合

### 组成

Spring 是一个框架，所以框架是支持很多功能的，我们要实现功能只需要打开配置即可。关于 Spring 框架支持的一些功能：

![1](https://drun1baby.top/2022/08/18/Spring%E5%BC%80%E5%8F%91%E5%AD%A6%E4%B9%A0/SpringWork.png)

- 最底下一定是核心
- 网上就是支持一些编程思想：IOC、AOP
- 支持 ORM（就是处理数据库等语句，可以用 MyBatis 来理解）
- 支持 Web ，比如 Web Application、Servlet 等

在 Spring 的官网上有一条学习路线：

![1](https://drun1baby.top/2022/08/18/Spring%E5%BC%80%E5%8F%91%E5%AD%A6%E4%B9%A0/SpringRoute.png)

由于 Spring 的配置非常繁琐，有时候我们不得不配置一些与 Web 应用本身关系不大的东西，所以出现 SpringBoot 来简化操作。

SpringBoot 是一个快速开发的脚手架，基于 SpringBoot 可以快速的开发单个微服务。

SpringCloud 是基于 SpringBoot 实现的。

## Spring 核心之 IOC

### IOC

新建项目并导入依赖：

```xml
<dependency>  
 <groupId>org.springframework</groupId>  
 <artifactId>spring-webmvc</artifactId>  
 <version>5.3.16</version>  
</dependency>  
  
<dependency>  
 <groupId>org.springframework</groupId>  
 <artifactId>spring-jdbc</artifactId>  
 <version>5.3.16</version>  
</dependency>
```

#### 传统的业务实现

在看 IOC 的编程思想之前先看看传统的编程思想：

传统的编程思想：`Controller`层写接口，去调用`Service`层，`Service`层里面有一个`Service`接口，还有一个`ServiceImpl`的实现类，具体业务就写在`ServiceImpl`里面。

`Service`层去调用`Dao`层，也就是实体类，`Dao`层有一个`Dao`的抽象接口，还有一个`DaoImpl`的实现类。

> 大致流程就是调用`Controller`来调用`Service`最后调用`Dao`。

先用代码实现一下传统编程思想。

**UserDao.java**

```java
package org.example.DAO;

public interface UserDao {
    public void getUser();
}
```

UserDao 的实现类——**UserDaoImpl.java**

```java
package org.example.DAO;

public class UserDaoImpl implements UserDao{
    @Override
    public void getUser() {
        System.out.println("输出获取用户数据");
    }
}
```

**UserService.java**——Service 业务层

```java
package org.example.Service;

public interface UserService {
    public void getUser();
}
```

**UserServiceImpl.java**——Service 业务实现类

```java
package org.example.Service;

import org.example.DAO.UserDao;
import org.example.DAO.UserDaoImpl;

public class UserServiceImpl implements UserService{

    private UserDao userDao = new UserDaoImpl();
    @Override
    public void getUser() {
        userDao.getUser();
    }
}
```

最后写一个启动的测试类：

```java
package org.example;

import org.example.Service.UserServiceImpl;

public class TestApplication {
    public static void main(String[] args) {
        UserServiceImpl userService = new UserServiceImpl();
        userService.getUser();
    }
}
```

#### IOC 的处理情景

我们这么写正常业务是没问题的，但是如果来了一个客户，他要求我们用 MySQL 获取 User，这里我们就需要把`UserServiceImpl`中的语句进行修改：

```java
private UserDAO userDAO = new UserDAOImpl();

// 修改如下

private UserDAO userDAO = new 对应的 DAOImpl 类
```

这样看着还行，只修改了一个地方，因为我们本质还是调用 DAO 层的东西，目前我们只有一个`getUser()`方法，后续会有更多的防火阀，像`getUserId()`、`getUserName()`等，这时候我们要是一个个换名字就太麻烦了，所以我们就会采用 IOC 的编程思想。

直接用代码来理解 IOC 这一编程思想。

我们在`UserServiceImpl`中加一个`setter`方法：

```java
package org.example.Service;

import org.example.DAO.UserDao;
import org.example.DAO.UserDaoImpl;

public class UserServiceImpl implements UserService{

    private UserDao userDao;

    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }

    @Override
    public void getUser() {
        userDao.getUser();
    }
}
```

再新建一个`MysqlUserDaoImpl`类去实现`UserDao`接口：

```java
package org.example.DAO;

public class MysqlUserDaoImpl implements UserDao{
    @Override
    public void getUser() {
        System.out.println("通过 MySQL 获取 User");
    }
}
```

然后修改一下启动类：

```java
package org.example;

import org.example.DAO.MysqlUserDaoImpl;
import org.example.Service.UserServiceImpl;

public class TestApplication {
    public static void main(String[] args) {
        UserServiceImpl userService = new UserServiceImpl();
        userService.setUserDao(new MysqlUserDaoImpl());
        userService.getUser();
    }
}
```

通过一个`setter`方法就非常有灵性了。

以前程序是主动创建对象，控制权在程序员手上，而使用`setter`注入之后程序员不再管理对象的创建，系统的耦合性就降低了，可以更专注于业务的实现。这就是 IOC 的原型，反转即把主动权交给用户。

#### IOC 本质

控制反转 IOC (Inversion of Control)，是一种设计思想，DI(依赖注入)是实现 IOC 的一种方法。在没有 IOC 的程序中，我们使用面向对象编程，对象的创建与对象间的依赖关系完全硬编码在程序中，对象的创建由程序自己控制，控制反转后将对象的创建交给了第三方，所谓控制反转就是获取依赖对象的方式反转了。

IOC 是 Spring 框架的核心内容，使用了多种方式完美实现了 IOC，可以使用 XML 配置，也可以使用注解，高版本的 Spring 也可以零配置实现 IOC。

Spring容器在初始化时先读取配置文件，根据配置文件或元数据创建与组织对象存入容器中，程序使用时再从Ioc容器中取出需要的对象。

![1](https://drun1baby.top/2022/08/18/Spring%E5%BC%80%E5%8F%91%E5%AD%A6%E4%B9%A0/IOCOrigin.png)

采用 XML 方式配置 Bean 的时候，Bean 的定义信息是和实现分离的，而采用注解的方式可以把两者合为一体，Bean 的定义信息直接以注解的形式定义在实现类中，从而达到了零配置的目的。

**控制反转是一种通过描述（XML或注解）并通过第三方去生产或获取特定对象的方式。在Spring中实现控制反转的是IoC容器，其实现方法是依赖注入（Dependency Injection,DI）。**

### Spring 通过 XML 进行装配

这是一个比较神奇的东西，通过读取 XML 文件来读取类确实比较神奇。

我们先编写一个 Hello 实体类：

```java
package com.solo.pojo;

public class Hello {
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
    
    public void show(){
        System.out.println("Hello, "+name);
    }
}
```

然后再写一个 beans.xml，用于我们 Spring 的装配：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--bean就是java对象，由Spring创建和管理-->
    <bean id="hello" class="com.solo.pojo.Hello">
        <property name="name" value="Spring"/>
    </bean>
</beans>
```

这里的`id`和`class`和 HTML 中很像，`id`只有一个，`class`就是对应我们要装配的类，`property`就是给对象中的属性赋值。

接着写一个测试代码：

```java
import com.solo.pojo.Hello;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class test {

    public static void main(String[] args) {
        //解析beans.xml文件，生成管理相应的Bean对象
        ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
        //getBean：参数即为spring配置文件中bean的id
        Hello hello = (Hello) context.getBean("hello");
        hello.show();
    }

}
```

#### 装配模式

- Hello 对象是由 Spring 对象创建的
- Hello 对象的属性是由 Spring 容器设置的

这个过程就叫控制反转：

- 控制：谁控制对象的创建。传统应用程序的对象是由程序本身控制创建的，使用 Spring 后，对象由 Spring 来创建
- 反转：程序本身不创建对象，而变成被动的接收对象。

IOC 是一种编程思想，由主动的编程变成被动的接收。

这是我们通过 XML 来获取类并装配。那么如果我们根据不同的业务进行装配呢？和之前的场景一样，有的业务需要我们从 MySQL 中读取数据，有的业务需要我们从 Oracle 中读取数据，我们应该如何实现。

#### 利用 IOC 的思想进行装配

回到之前的 IOC 的案例中，新建一个 beans2.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="mysqlImpl" class="org.example.DAO.MysqlUserDaoImpl"/>

    <bean id="UserServiceImpl" class="org.example.Service.UserServiceImpl">
        <!--ref引用spring中已经创建好的对象-->
        <!--value是一个具体的值，基本数据类型-->
        <property name="UserDao" ref="mysqlImpl"/>
    </bean>
</beans>
```

对应的启动器：

```java
package org.example;

import org.example.Service.UserServiceImpl;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class TestApplication {
    public static void main(String[] args) {
        //获取ApplicationContext，拿到spring容器
        ApplicationContext context = new ClassPathXmlApplicationContext("beans2.xml");
        //需要什么就直接get
        UserServiceImpl userServiceImpl = (UserServiceImpl) context.getBean("UserServiceImpl");
        userServiceImpl.getUser();
    }
}
```

假若这时候需要 OracleImpl 对象，则只需在 xml 配置文件中设置 `ref="oracleImpl"` 即可，无需去改动代码。

**总结**：

- 所有的类都要装配到 beans.xml 中
- 所有的 bean 都要通过容器去取
- 容器中取得的 bean，拿出来就是一个对象，用对象调用方法即可

**IOC 其实就是一句话：对象由 Spring 来创建、管理、装配。**

### IOC 创建对象的方式

#### 通过无参构造方法来创建

User.java：

```java
package org.example;

public class User {
    private String name;

    public User() {
        System.out.println("user无参构造方法");
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
    
    public void show(){
        System.out.println("name = "+name);
    }
}
```

beans.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="user" class="org.example.User">
        <property name="name" value="solo"/>
    </bean>
</beans>
```

测试类：

```java
import org.example.User;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class TestApplication {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
        //在执行getBean的时候，user已经创建好了，通过无参构造
        User user = (User) context.getBean("user");
        user.show();
    }
}
```

#### 通过有参构造方法来创建

```java
package org.example;

public class User {
    private String name;

    public User(String name) {
        this.name=name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void show(){
        System.out.println("name = "+name);
    }
}
```

beans.xml 有三种方式编写：

- 下标赋值
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
  
      <bean id="user" class="org.example.User">
          <constructor-arg index="0" value="solo"/>
      </bean>
  </beans>
  ```

- 类型赋值
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
  
      <bean id="user" class="org.example.User">
          <constructor-arg type="java.lang.String" value="solo"/>
      </bean>
  </beans>
  ```

- 直接通过参数名
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
  
      <bean id="user" class="org.example.User">
          <!--name指参数名-->
          <constructor-arg name="name" value="solo"/>
      </bean>
  </beans>
  ```

在配置文件加载的时候，其中管理的对象都已经初始化了，并且是单例模式，取到的对象是全局唯一。

## Spring 配置

先理解一下封装的思维，以及为什么要用 Spring 来配置。

在 Spring 当中有一个 bean.xml 叫做`applicationContext.xml`，它是用来统领所有配置的。

比如有三个程序员分别是张三、李四、王五。张三写了代码，把程序打包到`zhangsan.xml`，同理就有`lisi.xml`、`wangwu.xml`。这时候我们的`applicationContext.xml`来统领所有的配置，这样就非常符合我们的封装思想。

### 别名

`alias`设置别名，为`bean`设置别名，可以设置多个别名。

```xml
<!--设置别名：在获取Bean的时候可以使用别名获取-->
<alias name="user" alias="userNew"/>
```

别名的理解很简单，我们要调用的时候可以调用 name，也可以调用别名。

### Bean 的配置

- id：bean 的唯一标识符，也就是相当于我们的对象名
- class：bean 对象所对应的全限定名——包名 + 类型
- name：也是别名，而且 name 可以同时取多个别名

```xml
<bean id="user" class="com.example.pojo.User" name="u1 u2,u3;u4">

<property name="name" value="chen"/>

</bean>
```

### import

import 一般用于团队开发使用，它可以将多个配置文件导入合并为一个。

```xml
<import resource="beans.xm1"/>  <import resource="beans2.xml"/>  <import resource="beans3.xm1"/>
```

使用的时候直接使用总的配置就可以了。有重名的情况出现时，按照在总的 xml 中的导入顺序来进行创建，后导入的会重写先导入的，最终实例化的对象会是后导入 xml 中的那个。

## DI 依赖注入

### 构造注入

这就是上面所说的，其实就是对应的 IOC 业务情景。

### Set 方法注入

依赖注入：本质上就是 Set 注入

- 依赖：bean 对象的创建依赖于容器
- 注入：bean 对象中的所有属性由容器来注入

#### 实体类

```java
package pojo;

public class Address {
    private String address;

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }
}
```

```java
package pojo;

import java.util.*;

public class Student {
    private String name;
    private Address address;
    private String[] books;
    private List<String> hobbies;
    private Map<String,String> card;
    private Set<String> games;
    private String wife;
    private Properties info;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Address getAddress() {
        return address;
    }

    public void setAddress(Address address) {
        this.address = address;
    }

    public String[] getBooks() {
        return books;
    }

    public void setBooks(String[] books) {
        this.books = books;
    }

    public List<String> getHobbies() {
        return hobbies;
    }

    public void setHobbies(List<String> hobbies) {
        this.hobbies = hobbies;
    }

    public Map<String, String> getCard() {
        return card;
    }

    public void setCard(Map<String, String> card) {
        this.card = card;
    }

    public Set<String> getGames() {
        return games;
    }

    public void setGames(Set<String> games) {
        this.games = games;
    }

    public String getWife() {
        return wife;
    }

    public void setWife(String wife) {
        this.wife = wife;
    }

    public Properties getInfo() {
        return info;
    }

    public void setInfo(Properties info) {
        this.info = info;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", address=" + address +
                ", books=" + Arrays.toString(books) +
                ", hobbies=" + hobbies +
                ", card=" + card +
                ", games=" + games +
                ", wife='" + wife + '\'' +
                ", info=" + info +
                '}';
    }
}
```

#### XML 文件注入

beans.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="student" class="pojo.Student">
        <property name="name" value="solo"/>
    </bean>
</beans>
```

#### 测试类

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import pojo.Student;

public class TestApplication {

    public static void main(String[] args) {
        
        ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
        //在执行getBean的时候, user已经创建好了 , 通过无参构造
        Student student = (Student) context.getBean("student");
        //调用对象的方法 . 
        System.out.println(student.getName());
    }
}
```

官方文档上支持很多种注入方式：

![1](https://drun1baby.top/2022/08/18/Spring%E5%BC%80%E5%8F%91%E5%AD%A6%E4%B9%A0/beans.png)

#### Bean 注入

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="addr" class="pojo.Address">
        <property name="address" value="china"/>
    </bean>
    
    <bean id="student" class="pojo.Student">
        <property name="address" ref="addr"/>
    </bean>
</beans>
```

#### 数组注入

```xml
<property name="books">  
    <array>  
        <value>java</value>
        <value>python</value>
    </array>  
</property>
```

#### List 注入

```xml
<property name="hobbies">
    <list>
        <value>ai</value>
        <value>cybersecurity</value>
    </list>
</property>
```

#### Map 注入

```xml
<!--map键值对注入-->
<property name="card">
    <map>
        <entry key="username" value="root"/>
        <entry key="password" value="root"/>
    </map>
</property>
```

#### Set 注入

```xml
<property name="games">
    <set>
        <value>LOL</value>
        <value>starcrafts</value>
    </set>
</property>
```

#### 空指针 null 注入

```xml
<property name="wife">
    <null></null>
</property>
```

####  Properties 常量注入

```xml
<property name="info">
    <props>
        <prop key="id">114514</prop>
        <prop key="name">solo</prop>
    </props>
</property>
```

### 扩展方式注入

在官方文档中是 c 命名与 p 命名空间注入。

![1](https://drun1baby.top/2022/08/18/Spring%E5%BC%80%E5%8F%91%E5%AD%A6%E4%B9%A0/Other.png)

其实这里 p 就代表 Properties，c 就代表 Constructor。

#### 环境

新建一个实体类：

**User.java**

> 这里没有有参构造器，后续会看为什么需要加上有参构造器。

```java
package pojo;

public class User {
    private String name;
    private int age;

    public void setName(String name) {
        this.name = name;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

#### p 命名空间

p 命名空间注入：需要在头文件中加入约束文件，也就是一个命名空间。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd"
       xmlns:p="http://www.springframework.org/schema/p">
	<!--P命名空间，属性仍然要设置set方法-->
    <bean id="user" class="pojo.User" p:name="solo" p:age="19"/>
</beans>
```

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import pojo.User;

public class Test {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
        User user = (User) context.getBean("user");
        System.out.println(user);
    }
}
```

#### c 命名空间注入

也是在头文件中加上约束文件：

```xml
 导入约束 : 
xmlns:c="http://www.springframework.org/schema/c"
 <!--C(构造: Constructor)命名空间 , 属性依然要设置set方法-->
<bean id="user" class="com.example.pojo.User" c:name="Drunkbaby" c:age="20"/>
```

但是这里就需要加上有参构造器。

![1](https://drun1baby.top/2022/08/18/Spring%E5%BC%80%E5%8F%91%E5%AD%A6%E4%B9%A0/Constructor.png)

### Bean 的作用域

在 Spring 中，哪些组成应用程序的主体以及由 Spring IOC 容器所管理的对象被称为 bean。简单来说，**bean 就是由 IOC 容器初始化、装配以及管理的对象。**

![1](https://drun1baby.top/2022/08/18/Spring%E5%BC%80%E5%8F%91%E5%AD%A6%E4%B9%A0/bean.png)

几种作用域中，request、session 作用域仅在基于 Web 的应用中使用（不必关心你所采用的是什么 Web 应用框架），只能用在基于 Web 的 Spring ApplicationContext 环境。

#### 单例模式

单例模式是一种常用的软件设计模式，在它的核心结构中只包含一个被称作单例类的特殊类。通过单例模式可以保证系统中一个类只有一个实例而且这个实例易于外界访问，从而方便对实例个数的控制并节约资源。

```xml
<bean id="user" class="com.example.pojo.User"" c:name="cxk" c:age="19" scope="singleton"></bean>
```

#### 原型模式

每一次从容器中 get 的时候都会产生一个新对象。

```xml
<bean id="user" class="com.example.pojo.User"" c:name="cxk" c:age="19" scope="prototype"></bean>
```

## Bean 的自动装配

### 自动装配说明

由于在手动配置 XML 的过程中，常常会发生字母缺漏和大小写等错误，而无法对其进行检查，使开发效率降低。采用自动装配就可以避免这些错误，并且使配置简单化。

- 自动装配是使用 Spring 满足 bean 依赖的一种方法
- Spring 会在应用上下文中为某个 bean 寻找其依赖的 bean

Spring 中 bean 有三种装配机制：

1. 在 XML 中显式配置
2. 在 Java 中显式配置
3. 隐式的 bean 发现机制和自动装配

### 样例代码

```java
package pojo;

public class Cat {
    public void shout(){
        System.out.println("cat");
    }
}
```

```java
package pojo;

public class Dog {
    public void shout(){
        System.out.println("dog");
    }
}
```

```java
package pojo;

public class Person {
    private Cat cat;
    private Dog dog;
    private String name;

    @Override
    public String toString() {
        return "Person{" +
                "cat=" + cat +
                ", dog=" + dog +
                ", name='" + name + '\'' +
                '}';
    }

    public Cat getCat() {
        return cat;
    }

    public void setCat(Cat cat) {
        this.cat = cat;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Dog getDog() {
        return dog;
    }

    public void setDog(Dog dog) {
        this.dog = dog;
    }
}
```

编写 Spring 配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="cat" class="pojo.Cat"/>
    <bean id="dog" class="pojo.Dog"/>
    
    <bean id="person" class="pojo.Person">
        <property name="name" value="solo"/>
        <property name="cat" ref="cat"/>
        <property name="dog" ref="dog"/>
    </bean>
</beans>
```

测试：

```java
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import pojo.Person;

public class TestAuto {
    @Test
    public void Test(){
        ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
        Person person = context.getBean("person", Person.class);
        person.getCat().shout();
        person.getDog().shout();
    }
}
```

### byName 和 byType 自动装配

#### byName 自动装配

比较简单，修改`beans.xml`配置文件即可。

```xml
<bean id="person" class="pojo.Person" autowire="byName">
    <property name="name" value="solo"/>
</bean>
```

byName 形式的自动装配会在容器上下文中查找和自己对象`set`后面的值对应的`beanid`，就是我们要装配的`beanid`要和`setxxx`方法的`xxx`一样。

#### byType 自动装配

```xml
<bean id="person" class="pojo.Person" autowire="byType">
    <property name="name" value="solo"/>
</bean>
```

byType 形式的自动装配会在容器上下文中查找和自己对象属性类型相同的`bean`，也就是找`class`是不是一样的。

### 使用注解自动装配

JDK1.5 开始支持注解，Spring 2.5 开始全面支持注解。

在 Spring 配置文件中引入 context 文件头，可以添加两行配置代表支持注解的语句：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context  http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>
</beans>
```

#### 使用 @AutoWired 注解进行自动装配

`@AutoWired`注解可以代替`setter`方法，可以写在属性上方也可以写在`setter`方法上，写在属性上甚至可以直接忽略`setter`方法。

```java
package pojo;

import org.springframework.beans.factory.annotation.Autowired;

public class Person {

    @Autowired
    private Cat cat;
    @Autowired
    private Dog dog;
    private String name;

    @Override
    public String toString() {
        return "Person{" +
                "cat=" + cat +
                ", dog=" + dog +
                ", name='" + name + '\'' +
                '}';
    }

    public Cat getCat() {
        return cat;
    }

    public void setCat(Cat cat) {
        this.cat = cat;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Dog getDog() {
        return dog;
    }

    public void setDog(Dog dog) {
        this.dog = dog;
    }
}
```

**`@Nullable`**

这个字段标记了这个注解说明这个字段可以为空。

```java
public People(@Nullable String name) {  
    this.name = name;  
}
```

**`@Autowired(required=false) `**

如果显式定义了`AutoWired`的`required`属性为`false`，说明这个对象可以为`null`，否则不允许为空。

#### 使用 @AutoWired + @Qualifer

`@Qualifer`是用来指定`beanid`的，和 byName 比较像。当`@AutoWired`不能唯一装配式，需要使用`@Qualifer`来辅助。

我们可以修改一下配置文件的内容：

```xml
<bean id="dog2" class="com.example.pojo.Dog"/><bean id="cat2" class="com.example.pojo.Cat"/>
```

在没有`@Qualifer`的情况下会直接报错。

然后在属性上添加`@Qualifer`：

```java
@Autowired
@Qualifier(value = "cat2")
private Cat cat;
@Autowired
@Qualifier(value = "dog2")
private Dog dog;
```

#### @Resource 自动装配

这是一种非常灵活的自动注入，可以理解为`byName+byType`。

- `@Resource`如果有指定的`name`属性，先按照该属性进行`byName`方法查找装配
- 其次再进行默认的`byName`方式进行装配
- 如果以上都不成功，则按照`byType`的方式自动装配
- 都不成功则报异常

```xml
<bean id="dog1" class="com.example.pojo.Dog"/><bean id="dog2" class="com.example.pojo.Dog"/><bean id="cat1" class="com.example.pojo.Cat"/><bean id="cat2" class="com.example.pojo.Cat"/>
```

```java
public class People {
	@Resource(name="cat2")
	private Cat cat;
	@Resource(name="dog2")
	private Dog dog;
	private String name;
}
```

#### 区别

`@Autowired` 与 `@Resource` 异同：

- `@Autowired` 与 `@Resource` 都可以用来装配 bean。都可以写在字段上，或写在 set 方法上。
- `@Autowired` 通过 **byType** 的方式实现，而且必须要求这个对象存在。
- `@Resource` 默认通过 **byname** 的方式实现，如果找不到名字，则通过 byType 实现！如果两个都找不到的情况下，就报错。
- 它们的作用相同都是用注解方式注入对象，但执行顺序不同。`@Autowired` 先 byType，`@Resource`先 byName。

## 使用注解开发

### 说明

在 Spring 4 之后，想要使用注解形式必须要引入 aop 的包，这个在`Spring-webmvc`的包里面基本也是自带的

我们之前都是用 bean 的标签进行 bean 注入，但是在实际开发中，我们一般都会使用注解来开发。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context  http://www.springframework.org/schema/context/spring-context.xsd">

    <!--指定要扫描的包，这个包下的注解就会生效-->
    <context:component-scan base-package="pojo"/>
    
</beans>
```

在指定包下面写一个类，添加注解：

```java
package pojo;

import org.springframework.stereotype.Component;

// 相当于配置文件中 <bean id="user" class="当前注解的类"/>
@Component("user")
public class User {
    public String name = "solo";
}
```

测试：

```java
import org.springframework.context.support.ClassPathXmlApplicationContext;
import pojo.User;

public class Test {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
        User user = context.getBean("user", User.class);
        System.out.println(user.name);
    }
}
```

### 属性注入

这个东西其实就是赋值，可以不使用 setter 方法，直接利用注解赋值。

```java
package pojo;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class User {
    @Value("solo")
    public String name;
}
```

如果提供了 setter 方法，也可以在 setter 上面添加注解。

这个 `@Value` 的注解在 Spring-mvc 的项目会经常用到，而里面的这个值可以写在 `application.properties` 中，到时候就可以直接调用。

### 衍生注解

**@Component 三个衍生注解：**

为了更好地分层，Spring 可以使用其他三个注解。

- @Controller：Web 层
- @Service：Service 层
- @Repository：Dao 层

写上这些注解，就相当于把这个类交给 Spring 管理装配了。

### 自动装配注解

@Autowired：默认是 byType 方式，如果匹配不上，就会 byName

@Nullable：字段标记了这个注解，说明该字段可以为空

@Resource：默认是 byName 方式，如果匹配不上，就会 byType

### 作用域

**@scope**

- singleton：默认的，Spring 会采用单例模式创建这个对象。关闭工厂 ，所有的对象都会销毁。
- prototype：多例模式。关闭工厂 ，所有的对象不会销毁。内部的垃圾回收机制会回收

## 用 Java 的方式配置 Spring

这个到了 SpringBoot 里面可以说是主流的选择了，也就是我们平常写项目时的 `config` 文件夹，因为 Spring 本身写 XML 太累了，繁琐至极，所以用 Java 的方式写 config，可以使得我们代码的可读性更高。

实体类：

```java
package pojo;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

// 这里这个注解的意思,就是说明这个类被Spring接管了,注册到了容器中  
@Component
public class User {
    private String name;

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                '}';
    }

    public String getName() {
        return name;
    }

    @Value("solo")
    public void setName(String name) {
        this.name = name;
    }
}
```

配置文件：

```java
package config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import pojo.User;

//这个也会被Spring托管注册到容器中，因为它本身就是一个@Component
@Configuration
public class MyConfig {

    //注册一个bean就相当于我们之前写的一个bean标签
    //方法的名字就相当于bean标签的id属性，返回值就相当于class属性
    @Bean
    public User getUser(){
        return new User();   //就是返回要注入的bean对象
    }
}
```

测试类：

```java
import config.MyConfig;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import pojo.User;

public class Test {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(MyConfig.class);
        User user = (User) context.getBean("getUser");
        System.out.println(user.getName());
    }
}
```

## 代理模式与 AOP

### 动态代理

虽然之前已经学习过动态代理了，但是还是先回顾一下。

#### 简介

动态代理的代理类是动态生成的，不是我们直接写好的。动态代理分为两大类：基于接口的动态代理，基于类的动态代理。

- 基于接口：JDK 原生动态代理
- 基于类：cglib
- java 字节码实现：javassist

动态代理需要了解两个类：`Proxy`和`InvocationHandler`。

#### InvocationHandler

这是`java.lang.reflect`包下的一个接口。`InvocationHandler`是**由代理实例的调用处理程序实现**的接口，每个代理实例都有一个关联的调用处理程序。当在代理实例上调用方法时，方法调用将被编码并分派到其调用处理程序的`invoke`方法。

上面是官方的描述，下面我们解释一下。

*先理解什么是代理实例*。

假如我们有一个真实对象`RealSubject`，它有一些方法。现在我们想在调用它的方法前后加一些额外逻辑，比如打印日志、权限检查等，但是我们又不像修改原来的代码。这时候我们就可以创建一个**代理对象**放在我们和`RealSubject`中间，我们去调用代理对象的方法，而代理对象再去调用真实对象的方法，中间就可以插入额外逻辑。

但是在动态代理中代理对象是 JVM 在运行时**动态生成**的，我们没法直接给这个代理对象写一些额外逻辑，所以就需要我们提供一个“处理器”来告诉它应该怎么做。这个处理器就是`InvocationHandler`。**所以我们所有的额外逻辑都是放到处理器中去的**。

*接下来逐句理解这段话：*

- **由代理实例的调用处理程序实现的接口**：这个意思其实就是`InvocationHandler`是一个接口，我们需要写一个类来实现它，这个实现类的实例就是**调用处理程序**。每一个代理对象都绑定着一个这样的处理器用来放额外逻辑。
- **每个代理实例都有一个关联的调用处理程序**：当我们用`Proxy.newProxyInstance()`创建代理对象时，必须传入一个`InvocationHandler`实例，这个代理对象以后所有的方法调用都会交给这个处理器来处理。
- **当在代理实例上调用方法时，方法调用将被编码并分派到其调用处理程序的 invoke 方法**：这是最关键的一句话，意思就是我们调用代理对象的任何方法，这个调用不会直接执行，而是会被编码成一个`Method`对象和参数数组，然后统一交给处理器的`invoke`方法处理。

看一下样例代码：

```java
// 1. 定义接口
interface Hello {
    void sayHello(String name);
}

// 2. 真实对象
class HelloImpl implements Hello {
    public void sayHello(String name) {
        System.out.println("Hello, " + name);
    }
}

// 3. 调用处理程序：实现 InvocationHandler
class MyHandler implements InvocationHandler {
    private Object target;  // 真实对象

    public MyHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("方法调用前：记录日志");
        Object result = method.invoke(target, args);  // 调用真实对象的方法
        System.out.println("方法调用后：记录日志");
        return result;
    }
}

// 4. 创建代理并使用
public class Main {
    public static void main(String[] args) {
        Hello real = new HelloImpl();
        Hello proxy = (Hello) Proxy.newProxyInstance(
            Hello.class.getClassLoader(),
            new Class[]{Hello.class},
            new MyHandler(real)  // 关联调用处理程序
        );

        proxy.sayHello("World");
        // 输出：
        // 方法调用前：记录日志
        // Hello, World
        // 方法调用后：记录日志
    }
}
```

**总结：**代理对象其实就是一个”空壳“，它把所有方法调用都转发给我们的`InvocationHandler.invoke()`，我们在`invoke()`中决定要不要调用真实对象以及调用前后做什么。

#### Proxy

这也是`java.lang.reflect`包下的一个类，它提供创建动态代理类和实例的静态方法，它也是由这些方法创建的所有动态代理类的超类。

我们用`Proxy.newProxyInstance()`来创建类。

动态代理的底层其实还是反射。

动态代理的好处：

1. 可以使真实角色的操作更加纯粹，不用关注一些公共的业务
2. 公共的业务交给了代理角色，实现了业务的分工
3. 公共业务发生扩展的时候方便集中管理
4. 一个动态代理类代理的是一个接口，一般就是对应一类业务
5. 一个动态代理类可以代理多个类，只要是实现了同一个接口即可

### AOP

#### 什么是 AOP

AOP 意为面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP 是 OOP 的延续，是软件开发中的一个热点，也是 Spring 框架中的一个重要内容，是函数式编程的一种衍生范式。利用 AOP 可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合性降低，提高程序的可重用性，同时提高开发效率。

面向切面编程，实现在不修改源代码的情况下给程序动态统一添加额外功能的一种技术。

![1](https://i-blog.csdnimg.cn/blog_migrate/dbd3ffa3fe7d90dceb73a7af22a6281a.png)

AOP 可以拦截指定的方法并对方法进行增强，而且无需侵入到业务代码中，使业务和非业务处理逻辑分离，比如 Spring 的事务，通过事务的注解配置，Spring 会自动在业务方法中开启、提交业务，并且在业务处理失败时执行相应的回滚策略。

#### AOP 的作用

AOP 采取横向抽取机制（动态代理），取代了传统纵向继承机制的重复性代码，其应用主要体现在事务处理、日志管理、权限控制、异常处理等方面。

主要作用是分离功能性需求和非功能性需求，使开发人员可以集中处理某一个关注点或者横切逻辑，减少对业务代码的侵入，增强代码的可读性和可维护性。

简单来说，AOP 的作用就是保证开发者在不修改源代码的前提下为系统中的业务组件添加某种通用功能。

#### AOP 的应用场景

比如典型的 AOP 的应用场景：

- 日志系统
- 事务管理
- 权限验证
- 性能监测

#### Spring AOP 的术语

##### AOP 核心概念

| 名称                | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| Joinpoint（连接点） | 指那些被拦截到的点，在 Spring 中，指可以被动态代理拦截目标类的方法 |
| Pointcut（切入点）  | 指要对哪些 Joinpoint 进行拦截，即被拦截的连接点              |
| Advice（通知）      | 指拦截到 Joinpoint 之后要做的事情，即对切入点增强的内容      |
| Target（目标）      | 指代理的目标对象                                             |
| Weaving（植入）     | 指把增强代码应用到目标上，生成代理对象的过程                 |
| Proxy（代理）       | 指生成代理对象                                               |
| Aspect（切面）      | 切入点和通知的集合                                           |

**Spring AOP 通知分类：**

| 通知                           | 说明                               |
| ------------------------------ | ---------------------------------- |
| before（前置通知）             | 通知方法在目标方法调用之前执行     |
| after（后置通知）              | 通知方法在目标方法返回或异常后调用 |
| after-returning（返回后通知）  | 通知方法会在目标方法返回后调用     |
| after-throwing（抛出异常通知） | 通知方法会在目标方法抛出异常后调用 |
| around（环绕通知）             | 通知方法会将目标方法封装起来       |

**Spring AOP 织入时期：**

| 时期     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| 编译期   | 切面在目标类编译时被织入，这种方式需要特殊的编译器，AspectJ 的织入编译器就是以这种方式织入切面的 |
| 类加载期 | 切面在目标类加载到 JVM 时被织入，这种方式需要特殊的类加载器，它可以在目标类引入应用之前增强目标类的字节码 |
| 运行期   | 切面在应用运行的某个时期被织入，一般情况下，在织入切面时，AOP 容器会为目标对象动态创建一个代理对象，Spring AOP 采用的就是这种织入方式 |

### 使用 Spring 实现 AOP

使用 AOP 织入需要导入一个依赖包：

```xml
<!-- Source: https://mvnrepository.com/artifact/org.aspectj/aspectjweaver -->
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.4</version>
    <scope>runtime</scope>
</dependency>
```

#### 使用 Spring 的 API 接口

先写 Service 业务层：

```java
package service;

public interface UserService {
    public void add();
    public void delete();
    public void update();
    public void select();
}
```

```java
package service;

public class UserServiceImpl implements UserService{
    @Override
    public void delete() {
        System.out.println("删除用户");
    }

    @Override
    public void add() {
        System.out.println("增加用户");
    }

    @Override
    public void update() {
        System.out.println("更新用户");
    }

    @Override
    public void select() {
        System.out.println("查询用户");
    }
}
```

再写 log 增强层：

```java
package log;

import org.springframework.aop.MethodBeforeAdvice;
import org.springframework.lang.Nullable;

import java.lang.reflect.Method;

public class Log implements MethodBeforeAdvice {

    //method：要执行的目标对象的方法
    //args：参数
    //target：目标对象
    @Override
    public void before(Method method, Object[] args, @Nullable Object target) throws Throwable {
        System.out.println(target.getClass().getName()+"的"+method.getName()+"被执行了");
    }
}
```

```java
package log;

import org.springframework.aop.AfterReturningAdvice;
import org.springframework.lang.Nullable;

import java.lang.reflect.Method;

public class AfterLog implements AfterReturningAdvice {

    //returnValue：返回值
    @Override
    public void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable {
        System.out.println("执行了"+method.getName()+"返回结果为："+returnValue);
    }
}
```

然后写 Spring 配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean id="userService" class="service.UserServiceImpl"/>
    <bean id="log" class="log.Log"/>
    <bean id="afterLog" class="log.AfterLog"/>

<!--    配置aop：需要导入aop的约束-->
    <aop:config>
<!--        切入点：expression：表达式：execution(要执行的位置 * * * * *)-->
        <aop:pointcut id="pointcut" expression="execution(* service.UserServiceImpl.*())"/>

<!--        执行环绕增加-->
        <aop:advisor advice-ref="log" pointcut-ref="pointcut"/>
        <aop:advisor advice-ref="afterLog" pointcut-ref="pointcut"/>
    </aop:config>
</beans>
```

测试类：

```java
import org.springframework.context.support.ClassPathXmlApplicationContext;
import service.UserService;

public class Test {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
        UserService userService = context.getBean("userService", UserService.class);

        userService.add();
    }
}
```

这里`getBean`的时候要用接口，因为动态代理代理的是接口。

#### 使用自定义类

写一个自定义切入点：

```java
package diy;

public class DiyPointcut {

    public void before(){
        System.out.println("=============方法执行前=============");
    }

    public void after(){
        System.out.println("=============方法执行后=============");
    }
}
```

修改一下配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean id="userService" class="service.UserServiceImpl"/>
    <bean id="log" class="log.Log"/>
    <bean id="afterLog" class="log.AfterLog"/>

<!--&lt;!&ndash;    配置aop：需要导入aop的约束&ndash;&gt;-->
<!--    <aop:config>-->
<!--&lt;!&ndash;        切入点：expression：表达式：execution(要执行的位置 * * * * *)&ndash;&gt;-->
<!--        <aop:pointcut id="pointcut" expression="execution(* service.UserServiceImpl.*())"/>-->

<!--&lt;!&ndash;        执行环绕增加&ndash;&gt;-->
<!--        <aop:advisor advice-ref="log" pointcut-ref="pointcut"/>-->
<!--        <aop:advisor advice-ref="afterLog" pointcut-ref="pointcut"/>-->
<!--    </aop:config>-->

    <bean id="diy" class="diy.DiyPointcut"/>
    <aop:config>
<!--        自定义切面，ref要引用的类-->
        <aop:aspect ref="diy">
<!--            切入点-->
            <aop:pointcut id="pointcut" expression="execution(* service.UserServiceImpl.*(..))"/>
<!--            切面-->
            <aop:before method="before" pointcut-ref="pointcut"/>
            <aop:after method="after" pointcut-ref="pointcut"/>
        </aop:aspect>
    </aop:config>
</beans>
```

#### 使用注解

```java
package diy;

import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;

@Aspect //标注这个类是一个切面
public class AnnoPointCut {

    @Before("execution(* service.UserServiceImpl.*(..))")
    public void before(){
        System.out.println("=============方法执行前=============");
    }

    @After("execution(* service.UserServiceImpl.*(..))")
    public void after(){
        System.out.println("=============方法执行后=============");
    }
    
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean id="userService" class="service.UserServiceImpl"/>
    <bean id="log" class="log.Log"/>
    <bean id="afterLog" class="log.AfterLog"/>

    <bean id="annoPointCut" class="diy.AnnoPointCut"/>
<!--    开启注解支持-->
    <aop:aspectj-autoproxy/>

</beans>
```

## 整合 MyBatis

### 基础开发

先导入依赖：

```xml
<dependencies>  
    <dependency>  
        <groupId>junit</groupId>  
        <artifactId>junit</artifactId>  
        <version>4.12</version>  
    </dependency>  
    <dependency>  
        <groupId>org.mybatis</groupId>  
        <artifactId>mybatis</artifactId>  
        <version>3.5.2</version>  
    </dependency>  
    <dependency>  
        <groupId>mysql</groupId>  
        <artifactId>mysql-connector-java</artifactId>  
        <version>5.1.47</version>  
    </dependency>  
    <dependency>  
        <groupId>org.springframework</groupId>  
        <artifactId>spring-webmvc</artifactId>  
        <version>5.1.10.RELEASE</version>  
    </dependency>  
    <dependency>  
        <groupId>org.springframework</groupId>  
        <artifactId>spring-jdbc</artifactId>  
        <version>5.1.10.RELEASE</version>  
    </dependency>  
 	<dependency>  
        <groupId>org.aspectj</groupId>  
        <artifactId>aspectjweaver</artifactId>  
        <version>1.9.4</version>  
    </dependency>  
    <dependency>  
        <groupId>org.mybatis</groupId>  
        <artifactId>mybatis-spring</artifactId>  
        <version>2.0.2</version>  
    </dependency>  
</dependencies>
```

写一个实体类：

```java
package pojo;

public class User {
    private int id;
    private String name;
    private String pwd;

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", pwd='" + pwd + '\'' +
                '}';
    }
}
```

编写核心配置文件 mybatis-config.xml：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <typeAliases>
        <package name="pojo"/>
    </typeAliases>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/javaseclab?characterEncoding=utf8"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="UserMapper.xml"/>
    </mappers>
</configuration>
```

编写接口：

```java
package mapper;

import pojo.User;

import java.util.List;

public interface UserMapper {
    public List<User> selectUser();
}
```

编写对应的 Mapper XML 文件：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    <mapper namespace="mapper.UserMapper">

    <select id="selectUser" resultType="pojo.User">
        select * from javaseclab.sqli
    </select>

</mapper>
```

再写一个测试类：

```java
import mapper.UserMapper;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import pojo.User;

import java.io.InputStream;
import java.util.List;

public class Test {
    public static void main(String[] args) throws Exception{
        String resource="mybatis-config.xml";
        InputStream is = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(is);
        SqlSession sqlSession = sqlSessionFactory.openSession();

        UserMapper mapper = sqlSession.getMapper(UserMapper.class);

        List<User> userList = mapper.selectUser();
        for (User user : userList) {
            System.out.println(user);
        }

        sqlSession.close();
    }
}
```

### MyBatis-Spring

上面的基础开发是一个正常的 MyBatis 程序的应用，下面就是 MyBatis 和 Spring 做的一个整合，充分发挥 Spring 的特点。

#### 基础知识

在开始使用 MyBatis-Spring 之前，我们需要先熟悉 Spring 和 MyBatis 这两个框架和有关术语。

MyBatis-Spring 需要一下版本：

| MyBatis-Spring | MyBatis | Spring 框架 | Spring Batch | Java    |
| -------------- | ------- | ----------- | ------------ | ------- |
| 2.0            | 3.5+    | 5.0+        | 4.0+         | Java 8+ |
| 1.3            | 3.4+    | 3.2.2+      | 2.1+         | Java 6+ |

如果使用 maven 来构建，只需要在 pom.xml 中加入依赖即可：

```xml
<dependency>
   <groupId>org.mybatis</groupId>
   <artifactId>mybatis-spring</artifactId>
   <version>2.0.2</version>
</dependency>
```

要和 Spring 一起使用 MyBatis，需要在 Spring 应用上下文中定义至少两样东西：一个`SqlSessionFactory`和至少一个数据映射器类。

##### 关于 SqlSessionFactory

在 MyBatis-Spring 中，可使用`SqlSessionFactoryBean`来创建`SqlSessionFactory`，要配置这个工厂 bean，只需要把下面的代码放到 Spring 的 XML 配置文件中：

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
 <property name="dataSource" ref="dataSource" />
</bean>
```

`SqlSessionFactory`需要一个`DataSource`（数据源），这可以是任意的`DataSource`，只需要和配置其他 Spring 数据库连接一样配置它就可以了。

在基础的 MyBatis 用法中，是通过`SqlSessionFactoryBuilder`来创建`SqlSessionFactory`的，而在 MyBatis-Spring 中，则是使用`SqlSessionFactoryBean`来创建的。

在 MyBatis 中，我们可以通过`SqlSessionFactory`来创建`SqlSession`，一旦我们获得了一个 session 之后，我们就可以使用它来执行映射了的语句，提交或回滚连接，最后当不再需要它的时候可以关闭 session。

`SqlSessionFactory`有一个唯一的必要属性：用于 JDBC 的 DataSource。这可以是任意的 DataSource 对象，它的配置方法和其他 Spring 数据库连接是一样的。

一个常用的属性是`configLocation`，它用来指定 MyBatis 的 XML 配置路径，它在需要修改 MyBatis 的基础配置非常有用。通常基础配置指的是`<setting>`或`<typeAliases>`元素。

##### 关于 SqlSessionTemplate

`SqlSessionTemplate`是 MyBatis-Spring 的核心，作为`SqlSession`的一个实现，这意味着可以使用它无缝代替代码中已经在使用的`SqlSession`。

模板可以参与到 Spring 的事务管理中，并且由于其是线程安全的，可以供多个映射器类使用，我们应该总是用`SqlSessionTemplate`来替换 MyBatis 默认的`DefaultSqlSession`实现，在同一应用程序中的不同类之间混杂使用可能会引起数据一致性的问题。

可以使用`SqlSessionFactory`作为构造方法的参数来创建`SqlSessionTemplate`对象。

#### 整合实现一

先写一个 Spring 配置文件 spring-dao.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
<!--    数据源-->
    <bean id="datasource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/javaseclab?characterEncoding=utf8"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
    </bean>

<!--    SqlSessionFactory-->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="datasource" />
        <property name="configLocation" value="classpath:mybatis-config.xml"/>
        <property name="mapperLocations" value="classpath:UserMapper.xml"/>
    </bean>

    <bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
<!--        只能用构造器注入，因为没有setter-->
        <constructor-arg index="0" ref="sqlSessionFactory"/>
    </bean>

    <bean id="userMapper" class="mapper.UserMapperImpl">
        <property name="sqlSession" ref="sqlSession"/>
    </bean>
</beans>
```

其中配置了 MyBatis 的数据源、`SqlSessionFactory`关联 MyBatis、注册`SqlSessionTemplate`，同时增加了 DAO 接口的实现类。

```java
package mapper;

import org.mybatis.spring.SqlSessionTemplate;
import pojo.User;

import java.util.List;

public class UserMapperImpl implements UserMapper{
    //Spring中都使用SqlSessionTemplat

    private SqlSessionTemplate sqlSession;

    public void setSqlSession(SqlSessionTemplate sqlSession) {
        this.sqlSession = sqlSession;
    }

    @Override
    public List<User> selectUser() {
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        return mapper.selectUser();
    }
}
```

加上测试类：

```java
import mapper.UserMapper;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import pojo.User;

public class Test {
    public static void main(String[] args) throws Exception{
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring-dao.xml");
        UserMapper userMapper = context.getBean("userMapper", UserMapper.class);
        for (User user : userMapper.selectUser()) {
            System.out.println(user);
        }
    }
}
```

#### 整合实现二

##### SqlSessionDaoSupport

`SqlSessionDaoSupport`是一个抽象的支持类，用来提供`SqlSession`，调用`getSqlSession()`方法能得到一个`SqlSessionTemplate`，之后可以执行 SQL 语句。比如：

```java
public class UserDaoImpl extends SqlSessionDaoSupport implements UserDao {
  public User getUser(String userId) {
    return getSqlSession().selectOne("org.mybatis.spring.sample.mapper.UserMapper.getUser", userId);
  }
}
```

在这个类中，通常更倾向于使用`MapperFactoryBean`，因为它不需要额外的代码。但是如果需要在 DAO 中做其他非 MyBatis 的工作或需要一个非抽象的实现类，那么这个类就很有用了。

`SqlSessionDaoSupport`需要通过属性设置一个`SqlSessionFactory`或`SqlSessionTemplate`，如果两个属性都被设置，那么`SqlSessionFactory`就会被忽略。

假设类`UserMapperImpl2`是`SqlSessionDaoSupport`的子类，可以加上 Spring 的配置来执行：

```xml
<bean id="UserMapper2" class="mapper.UserMapperImpl2">
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

```java
package mapper;

import org.apache.ibatis.session.SqlSession;
import org.mybatis.spring.support.SqlSessionDaoSupport;
import pojo.User;

import java.util.List;

public class UserMapperImpl2 extends SqlSessionDaoSupport implements UserMapper{
    @Override
    public List<User> selectUser() {
        SqlSession sqlSession = getSqlSession();
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        return mapper.selectUser();
    }
}
```

再修改一下测试文件即可。

## 声明式事务

### 事务

- 事务在项目开发过程中非常重要，涉及到数据的一致性的问题。
- 事务管理是企业级应用程序开发中必备技术，用来确保数据的完整性和一致性。

事务就是把一系列的动作当成一个独立的工作单元，这些动作要么全部完成，要么全部不起作用。

#### 事务的四个属性 ACID

- 原子性（atomicity）

事务是原子性操作，由一系列动作组成，事务的原子性确保动作要么全部完成，要么完全不起作用。

- 一致性（consistency）

一旦所有事务动作完成，事务就要被提交。数据和资源处于一种满足业务规则的一致性状态中。

- 隔离性（isolation）

可能多个事务会同时处理相同的数据，因此每个事务都应该与其他事务隔离开来，防止数据损坏。

- 持久性（durability）

事务一旦完成，无论系统发生什么错误，结果都不会受到影响。通常情况下，事务的结果被写到持久化存储器中。

#### 测试

我们在之前的案例中给`UserMapper`接口加上三个方法：

```java
//添加一个用户
int addUser(User user);
 
//根据id删除用户
int deleteUser(int id);
 
public List<User> test();
```

修改一下实现类：

```java
public class UserMapperImpl extends SqlSessionDaoSupport implements UserMapper {
 
   //增加一些操作
   public List<>User test() {
       UserMapper mapper = getSqlSession().getMapper(UserMapper.class);
       User user = new User(5,"小明","123456");
       mapper.addUser(user);
       mapper.deleteUser(5);
       return mapper.selectUser();
   } 
 
   public List<User> selectUser() {
       UserMapper mapper = getSqlSession().getMapper(UserMapper.class);
       return mapper.selectUser();
  }
 
   //新增
   public int addUser(User user) {
       UserMapper mapper = getSqlSession().getMapper(UserMapper.class);
       return mapper.addUser(user);
  }
   //删除
   public int deleteUser(int id) {
       UserMapper mapper = getSqlSession().getMapper(UserMapper.class);
       return mapper.deleteUser(id);
  }
 
}
```

在 UserMapper.xml 配置文件中添加映射，并把 delete 语句写错：

```xml
<insert id="addUser" resultType="User">
    insert into mybatis.user (id, name, pwd) values (#{id},#{name},#{pwd})
</insert>
 
<delete id="deleteUser" resultType="User">
    deletes from mybatis.user where id=#{id}
</delete>
 
<select id="test" resultType="User"/>
```

测试一下：

```java
@Test
public void test(){
    ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
    UserMapper mapper = (UserMapper) context.getBean("userMapper");
    List<User> userlist = mapper.test();
    for (User user : userlist) {
        System.out.println(user);
    }
}
```

最后报错，但是我们的插入操作成功了，这就是因为没有进行事务的管理。

#### Spring 的事务管理

Spring 在不同的事务管理 API 上定义了一个抽象层，使得开发人员不必了解底层的事务管理 API 就可以使用 Spring 的事务管理机制。Spring 支持编程式事务管理和声明式事务管理。

**编程式事务管理**

- 将事务管理代码嵌入到业务方法中来控制事务的提交和回滚
- 缺点：必须在每个事务操作业务逻辑中包含额外的事务管理代码

**声明式事务管理**

- 一般情况下比编程式事务好用
- 将事务管理代码从业务方法中分离出来，以声明的方式来实现事务管理
- 将事务管理作为横切关注点，通过 AOP 方法模块化。Spring 中通过 Spring AOP 框架支持声明式事务管理

**使用 Spring 管理事务，需要进行约束导入：tx**

```xml
xmlns:tx="http://www.springframework.org/schema/tx"
 
http://www.springframework.org/schema/tx
http://www.springframework.org/schema/tx/spring-tx.xsd
```

**事务管理器**

- 不管使用 Spring 的哪种事务管理策略事务管理器都是必需的
- 事务管理器就是 Spring 的核心事务管理抽象，管理封装了一组独立于技术的方法

**JDBC 事务**

```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
       <property name="dataSource" ref="dataSource" />
</bean>
```

**配置好事务管理器之后我们要配置事务的通知**

```xml
<!--配置事务通知-->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
   <tx:attributes>
       <!--配置哪些方法使用什么样的事务,配置事务的传播特性-->
       <tx:method name="add" propagation="REQUIRED"/>
       <tx:method name="delete" propagation="REQUIRED"/>
       <tx:method name="update" propagation="REQUIRED"/>
       <tx:method name="search*" propagation="REQUIRED"/>
       <tx:method name="get" read-only="true"/>
       <tx:method name="*" propagation="REQUIRED"/>
   </tx:attributes>
</tx:advice>
```

**配置 AOP**

```xml
<!--配置aop织入事务-->
<aop:config>
   <aop:pointcut id="txPointCut" expression="execution(* com.example.mapper.*.*(..))"/>
   <aop:advisor advice-ref="txAdvice" pointcut-ref="txPointCut"/>
</aop:config>
```

最终的配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/tx
       http://www.springframework.org/schema/tx/spring-tx.xsd
       http://www.springframework.org/schema/aop
       https://www.springframework.org/schema/aop/spring-aop.xsd">
 
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/mybatis?useSSL=true&amp;useUnicode=true&amp;characterEncoding=utf8"/>
        <property name="username" value="root"/>
        <property name="password" value=""/>
    </bean>
 
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource" />
        <!--绑定MyBatis配置文件-->
        <property name="configLocation" value="classpath:mybatis-config.xml"/>
        <property name="mapperLocations" value="classpath:com/example/mapper/UserMapper.xml"/>
    </bean>
 
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <constructor-arg ref="dataSource" />
    </bean>
 
    <!--结合AOP实现事务的织入-->
    <!--配置事务的类：-->
    <!--配置事务通知：-->
    <tx:advice id="txAdvice" transaction-manager="transactionManager">
        <!--给哪些方法配置事务-->
        <!--配置事务的传播特性-->
        <tx:attributes>
            <tx:method name="add" propagation="REQUIRED"/>
            <tx:method name="delete" propagation="REQUIRED"/>
            <tx:method name="query" read-only="true"/>
            <tx:method name="*" propagation="REQUIRED"/>
        </tx:attributes>
    </tx:advice>
 
    <!--配置事务切入-->
    <aop:config>
        <aop:pointcut id="txPointCut" expression="execution(* com.example.mapper.*.*(..))"/>
        <aop:advisor advice-ref="txAdvice" pointcut-ref="txPointCut"/>
    </aop:config>
</beans>
```

然后重新测试，这一次仍然报错都是插入也不成功了。

#### 为什么要配置事务

1. 如果不配置事务，可能存在数据提交不一致的情况
2. 如果不在 Spring 中配置声明式事务，我们就需要在代码中手动配置事务
3. 事务在项目的开发中非常重要，涉及到数据的一致性和完整性问题













