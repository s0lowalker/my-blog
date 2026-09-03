---
title: Fastjson基础
date: 2026-09-03
---

# Fastjson 基础

## 简介

Fastjson 是阿里巴巴用 Java 开发的高性能 JSON 库，用于将数据在 JSON 和 Java 对象之间互相转换。

主要使用两个接口来分别实现序列化和反序列化：

- `JSON.toJSONString()`将 Java 对象转化为 JSON 对象，这是序列化的过程
- `JSON.parseObject()/JSON.parse()`将 JSON 对象转化为 Java 对象，这是反序列化的过程

可以把 JSON 简单理解为一个字符串。

## 代码样例

### 序列化

先在 pom.xml 里导入 Fastjson 的依赖：

```xml
<dependency>  
 <groupId>com.alibaba</groupId>  
 <artifactId>fastjson</artifactId>  
 <version>1.2.24</version>  
</dependency>
```

定义一个 Student 类：

```java
package org.example;

public class Student {
    private String name;
    private int age;

    public Student() {
        System.out.println("构造函数");
    }

    public String getName() {
        System.out.println("getname");
        return name;
    }

    public void setName(String name) {
        System.out.println("setname");
        this.name = name;
    }

    public int getAge() {
        System.out.println("getage");
        return age;
    }

    public void setAge(int age) {
        System.out.println("setage");
        this.age = age;
    }
}
```

然后进行序列化，用`JSON.toJsonString()`来序列化对象：

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.serializer.SerializerFeature;

public class StudentSerialize {
    public static void main(String[] args) {
        Student student = new Student();
        student.setName("solo");

        String jsonString = JSON.toJSONString(student, SerializerFeature.WriteClassName);
        System.out.println(jsonString);
    }
}
```

序列化的逻辑可以调试看一下。

首先进入 JSON 类，然后进入`toJSONString()`方法，创建了一个`SerializeWriter`对象，序列化这一步在这里就已经完成了。

![1](https://drun1baby.top/2022/08/04/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8701-Fastjson%E5%9F%BA%E7%A1%80/SerializeWriter.png)

在下面的变量里有个 static 的变量，写着`members of JSON`，主要是一个值`DEFAULT_TYPE_KEY`为`@type`，这个在反序列化漏洞里很重要。

![2](https://drun1baby.top/2022/08/04/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8701-Fastjson%E5%9F%BA%E7%A1%80/static.png)

跟进到`SerializeWriter`里，里面定义了一些初值，然后把这个对象赋值给`out`作为后续`JSONSerializer`的参数。然后就是`toString()`方法显示最后的运行结果。

很明显`String jsonString = JSON.toJSONString(student, SerializerFeature.WriteClassName)`这个语句是最关键的。关注一下它的参数：

第一个参数是 student，就是我们要进行序列化的对象。

第二个参数是`SerializerFeature.WriteClassName`，这是`JSON.toJSONString()`中的一个设置属性值，设置之后在序列化的时候会多写入一个`@type`，即写上被序列化的类名，`type`可以指定反序列化的类，并且调用其`getter`/`setter`/`is`方法。

> Fastjson 接收的 JSON 可以通过 @type 字段来指定该 JSON 应该还原成什么类型的对象，方便反序列化的时候的操作。

输出结果：

```java
//设置了SerializerFeature.WriteClassName
构造函数
setname
setage
getage
getname
{"@type":"org.example.Student","age":18,"name":"solo"}

//不设置SerializerFeature.WriteClassName
构造函数
setname
setage
getage
getname
{"age":18,"name":"solo"}
```

### 反序列化

调用`JSON.parseObject()`来进行反序列化。

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;

public class StudentUnserialize01 {
    public static void main(String[] args) {
        String jsonString = "{\"@type\":\"org.example.Student\",\"age\":18,\"name\":\"solo\"}";

        Student student = JSON.parseObject(jsonString, Student.class, Feature.SupportNonPublicField);
        System.out.println(student);
        System.out.println(student.getClass().getName());
    }
}
```

输出结果：

```text
构造函数
setage
setname
org.example.Student@4b85612c
org.example.Student
```

## 其他基础知识

### 反序列化时的 Feature.SupportNonPublicField 参数

在反序列化的时候，如果不加`Feature.SupportNonPublicField`并且没有对应的`setter`，我们无法获取`age`的值，因为是私有属性。

如果想要还原出私有属性的话还需要在`JSON.parseObject()/JSON.parse()`中加上`Feature.SupportNonPublicField`参数。

修改一下 Student 类，把私有属性 age 的 setAge() 方法注释掉：

```text
//没添加Feature.SupportNonPublicField
构造函数
setname
org.example.Student@5c29bfd
org.example.Student
getname
solo
getage
0

//添加了Feature.SupportNonPublicField
构造函数
setname
org.example.Student@4b85612c
org.example.Student
getname
solo
getage
18
```

针对私有属性，Fastjson 有两种处理方法：

1. 该私有属性有公共的`setter`，即使不开启`Feature.SupportNonPublicField`也能成功给属性赋值，因为 Fastjson 在这里是调用的`setter`方法来赋值。
2. 该属性没有对应的`setter`方法，也无法通过其他方式（如构造方法）赋值时，Fastjson 默认无法对其进行反序列化。此时就需要开启`Feature.SupportNonPublicField`，让 Fastjson 能够直接通过反射访问私有字段来进行赋值。

### 只进行 JSON.parseObject(jsonString)

再看一下`parseObject()`的指定或不指定反序列化类型之间的差异。

由于 Fastjson 反序列化漏洞利用只和包含了`@type`的 JSON 数据有关，因此这里我们只对序列化时设置了`SerializerFeature.WriteClassName`，即含有`@type`指定反序列化类型的 JSON 数据进行反序列化。其实也存在`@type`不指定反序列化类型的攻击方式，这个后面再讨论。

修改一下 Student 类：

```java
package org.example;

import java.util.Properties;

public class Student {
    private String name;
    private int age;
    private String address;
    private Properties properties;

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

    public String getAddress() {
        System.out.println("getAddress");
        return address;
    }

    public Properties getProperties() {
        System.out.println("getProperties");
        return properties;
    }
}
```

再改一下反序列化，先默认调用`parseObject()`不带指定类型的参数：

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;

public class StudentUnserialize02 {
    public static void main(String[] args) {
        String jsonString="{\"@type\":\"org.example.Student\",\"age\":19," +
                "\"name\":\"solo\",\"address\":\"china\",\"properties\":{}}";

        JSONObject obj = JSON.parseObject(jsonString);
        System.out.println(obj);
        System.out.println(obj.getClass().getName());
    }
}
```

输出：

```text
构造函数
setName
getProperties
getAddress
getAge
getName
getProperties
{"name":"solo","age":0}
com.alibaba.fastjson.JSONObject
```

输出可以看到调用了 Student 类的构造函数、所有属性的`getter`方法、存在的`setter`方法，其中`getProperties()`调用了两次，最后可以看到是未反序列化成功的。

然后给反序列化语句加上指定反序列化的类型，就能得到成功反序列化的回显。

```text
构造函数
setName
getProperties
org.example.Student@46f7f36a
org.example.Student
```

### parse 和 parseObject 的区别

两者的主要区别就是`parseObject()`返回的是`JSONObject`而`parse()`返回的是实际类型的对象。当在没有对应类的定义的情况下，一般情况下都会使用`JSON.parseObject()`来获取数据。

> Fastjson 中的`parse()`和`parseObject()`方法都可以用来将 JSON 字符串反序列化成 Java 对象，`parseObject()`本质上也是调用了`parse()`进行反序列化的，但是`parseObject()`会额外把 Java 对象转化为 JSONObject 对象，即进行`JSON.toJSON()`。所以进行反序列化时的细节区别在于，`parse()`会识别并调用目标类的`setter`方法以及某些特殊条件的`getter`方法，而`parseObject()`由于多执行了`JSON.toJSON(obj)`，所以在处理过程中会调用反序列化目标类的所有`setter`和`getter`方法。

也就是说，我们用`parse()`反序列化会直接得到特定的类，而无需像`parseObject()`一样返回的是 JSONObject 类型的对象，还可能需要去设置第二个参数指定特定的类。

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.alibaba.fastjson.parser.Feature;

public class StudentUnserialize02 {
    public static void main(String[] args) {
        String jsonString="{\"@type\":\"org.example.Student\",\"age\":19," +
                "\"name\":\"solo\",\"address\":\"china\",\"properties\":{}}";

        //Student obj = JSON.parseObject(jsonString, Student.class);
        Object obj = JSON.parse(jsonString, Feature.SupportNonPublicField);
        System.out.println(obj);
        System.out.println(obj.getClass().getName());
    }
}
```

输出：

```text
构造函数
setName
getProperties
org.example.Student@41906a77
org.example.Student
```

## Fastjson 反序列化漏洞原理

Fastjson 在反序列化的时候会去找我们在`@type`中规定的类是哪个类，然后在反序列化的时候会自动调用这些`setter`与`getter`方法，但并不是所有的`setter`和`getter`。

**Fastjson 会对满足下列要求的 setter/getter 方法进行调用：**

满足条件的 setter：

- 非静态函数
- 返回类型为 void 或当前类
- 参数个数为1个

满足条件的 getter：

- 非静态方法
- 无参数
- **返回值类型继承自 Collection/Map/AtomicBoolean/AtomicInteger/AtomicLong**

Fastjson 的攻击其实是比较简单的，因为并没有那么多复杂的链子，也不需要反射修改值，直接在 json 中赋值就行了。

### 漏洞原理

前面的分析可知，Fastjson 是自己实现了一套序列化和反序列化的机制，并不是用的 Java 原生的序列化和反序列化的机制。在 Fastjson 的绝大部分版本中，Fastjson 的原理都是一样的，只不过不同版本是针对不同的黑名单或者利用不同利用链来进行绕过利用罢了。

通过 Fastjson 反序列化漏洞，攻击者可以传入一个恶意构造的 JSON 数据，程序对其进行反序列化后得到恶意类并执行了恶意类中的恶意函数，进而导致代码执行。

#### 如何反序列化出恶意类

从前面的代码样例可以知道，Fastjson 使用`parseObject()`/`parse()`进行反序列化的时候可以指定类型，如果指定的类型太大，包含太多子类，就有利用空间了。比如说，如果指定类型是`Object`或者`JSONObject`，则可以反序列化出来任意类。例如代码写`Object o = JSON.parseObject(poc,Object.class)`就可以反序列化出Object类或其任意子类，而Object又是任意类的父类，所以就可以反序列化出所有类。

#### 如何让触发反序列化得到的恶意类中的恶意函数

在某些情况下进行反序列化时会将反序列化得到的类的构造函数、`getter`、`setter`执行一遍，如果这三种方法中存在危险操作，则可能导致反序列化漏洞的存在。换句话说，就是攻击者传入要进行反序列化的类中的构造函数、`getter`方法、`setter`方法中要存在漏洞才能触发。

我们到`DefaultJSONParser.parseObject(Map object, Object fieldName)`中看下，JSON中以@type形式传入的类的时候，调用`deserializer.deserialize()`处理该类，并去调用这个类的`setter`和`getter`方法：

```java
public final Object parseObject(final Map object, Object fieldName) {
    ...
    // JSON.DEFAULT_TYPE_KEY即@type
    if (key == JSON.DEFAULT_TYPE_KEY && !lexer.isEnabled(Feature.DisableSpecialKeyDetect)) {
        ...
        ObjectDeserializer deserializer = config.getDeserializer(clazz);
        return deserializer.deserialze(this, clazz, fieldName);
```

若反序列化指定类型的类如`Student obj = JSON.parseObject(text, Student.class);`，该类本身的构造函数、`setter`方法、`getter`方法存在危险操作，则存在Fastjson反序列化漏洞。

若反序列化未指定类型的类如`Object obj = JSON.parseObject(text, Object.class);`，该若该类的子类的构造方法、`setter`方法、`getter`方法存在危险操作，则存在Fastjson反序列化漏洞。

## 总结

总结一下漏洞发生在反序列化的点，也就是 `Obj.parse` 和 `Obj.parseObject` 这里。必须的是传参要带入 class 的参数，最好带上 Feature.SupportNonPublicField。PoC 是通过 String 传进去的，要以 `@type` 打头。漏洞的原因是反序列化的时候去调用了 getter 和 setter 的方法。其余就没什么了，比较简单。

























