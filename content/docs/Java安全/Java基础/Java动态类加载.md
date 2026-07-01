---
title: java类的动态加载
date: 2026-04-24
tags: 
  - java安全
  - java类加载
categories: 
  - Web安全
  - java安全 
excerpt: java类加载初探
---

# java类的动态加载

## 类加载器和双亲委派机制

### 类加载器的作用

一句话概括：把类的字节码文件（.class）加载到 JVM 内存中，并转换成可以被虚拟机直接使用的 java 类，即 Class 对象。

- 加载 Class 文件

```java
Person p = new Person();
```

`Person`这个类本身是抽象的，通过`new`操作，实例化。类加载做的则是把类这个模板经过编译形成的字节码传入 JVM，生成一个具备这个类的所有数据的 Class 对象。(Class 对象就当成是这个类理解即可)

ClassLoader 的工作流程：

首先，一个类编译后形成的 class 文件经过类加载器进入 JVM并产生 Class 对象，然后在 JVM 中进行初始化，我们可以通过这个 Class 对象的 `getClassLoader()` 方法获取这个类对应的类加载器。我们也可以通过对这个 Class 对象调用其 `newInstance()` 方法来实例化一个对象，通过这个实例化对象的 `getClass()` 方法来获取对应的 Class 对象。

### 3种类加载器

#### 启动类加载器 (Bootstrap ClassLoader)

其底层原生代码属是由 C++ 编写的，属于 JVM 的一部分。

不继承于`java.lang.ClassLoader`类，也没有父类加载器，主要负责加载核心 java 库（即 JVM 本身），储存在`/jre/lib/rt.jar`中。

#### 扩展类加载器 (ExtensionsClassLoader)

这个类加载器是由`sun.misc.Launcher$ExtClassLoader`类实现的，用来加载`/jre/lib/ext`或`java.ext.dirs`指定的目录。Java 虚拟机会提供一个扩展库目录，此类加载器在目录中查找并加载 java 类。

在 JDK9 以及更高版本，由于引入了模块化系统后，此类加载器被替代为平台类加载器，加载 Java SE 平台模块（如 java.sql，java.xml 等）。

#### 应用类加载器

此加载器由`sun.misc.Launcher$AppClassLoader`实现，一般通过`java.class.path`或者`Classpath`环境变量来加载 java 类，也就是常说的 classpath 路径。一般我们使用这个加载器来加载 java 应用类，可以通过 `ClassLoader.getSystemClassLoader()`来获取它。

### 双亲委派机制

一句话概括：一个类加载器收到加载请求时，不是先自己尝试加载，而是把请求委派给父加载器，一层层向上传递，直到启动类加载器。只有当父加载器反馈不加载时，子加载器才会自己尝试加载。

#### 从错误的方向看双亲委派

```java
package java.lang;

public class String {

    public String toString(){
        return "hello";
    }
    public static void main(String[] args) {
        String s = new String();
        s.toString();
    }
}
```

这样看似乎没有问题，但是报错很神奇：

```text
错误: 在类 java.lang.String 中找不到 main 方法, 请将 main 方法定义为:
   public static void main(String[] args)
否则 JavaFX 应用程序类必须扩展javafx.application.Application
```

明明以及定义了 main 方法，却报错没有 main 方法。这里则涉及双亲委派机制。

类加载器在被调用时，也就是在`new`之前，是以启动类到扩展类到应用类加载器去寻找，而在启动类加载器中是能找到`java.lang.String`类的，这个`String`类里是没有 main 方法的。

### 代码块加载顺序

这里所说的代码块主要是四种：

- 静态代码块：`static{}`
- 构造代码块：`{}`
- 无参构造器：`classname()`
- 有参构造器：`classname(参数)`

#### 实例化对象

Person.java：

```java
package LoadClass;

public class Person {
    public String name;
    private int age;
    public static int id;

    static{
        System.out.println("静态代码块");
    }

    public static void staticFunction(){
        System.out.println("静态方法");
    }

    {
        System.out.println("构造代码块");
    }

    public Person() {
        System.out.println("无参构造方法");
    }

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
        System.out.println("有参构造方法");
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

Loadclasstest.java：

```java
package LoadClass;

public class Loadclasstest {
    public static void main(String[] args) throws Exception{
        Person person = new Person();
    }
}
```

运行结果：

```text
静态代码块
构造代码块
无参构造方法
```

类的实例化其实就是一种初始化，而类加载是在初始化之前触发的，静态代码块和静态方法是和类挂钩的，所以静态代码块先触发。然后不管调用的什么构造方法，都会先调用构造代码块，然后调用对应的构造方法。

#### 调用静态方法

Loadclasstest.java：

```java
package LoadClass;

public class Loadclasstest {
    public static void main(String[] args) throws Exception{
        Person.staticFunction();
    }
}
```

运行结果：

```text
静态代码块
静态方法
```

直接调用静态方法，会先调用静态方法块，然后调用对应的静态方法。

#### 对静态成员变量赋值

Loadclasstest.java：

```java
package LoadClass;

public class Loadclasstest {
    public static void main(String[] args) throws Exception{
        Person.id = 1;
    }
}
```

运行结果：

```text
静态代码块
```

对静态成员变量赋值涉及到类初始化的过程，所以静态代码块会在静态成员变量赋值时执行。

#### class获取类

```java
package LoadClass;

public class Loadclasstest {
    public static void main(String[] args) throws Exception{
        Class p = Person.class;
    }
}
```

运行结果为空，说明利用`class`获取类不会记载类。

#### forName 获取类

```java
package LoadClass;

public class Loadclasstest {
    public static void main(String[] args) throws Exception{
        Class.forName("LoadClass.Person");
    }
}          //执行静态代码块
```

```java
package LoadClass;

public class Loadclasstest {
    public static void main(String[] args) throws Exception{
        Class.forName("LoadClass.Person",true,ClassLoader.getSystemClassLoader());
    }
}                //执行静态代码块
```

```java
package LoadClass;

public class Loadclasstest {
    public static void main(String[] args) throws Exception{
        Class.forName("LoadClass.Person",false,ClassLoader.getSystemClassLoader());
    }
}                 //没有输出
```

这里的`Class.forName()`是经过重载的，第二个参数是是否进行初始化，静态代码块是在类初始化时执行的，所以会有不同的结果。

#### ClassLoader.loadClass()获取类

```java
package LoadClass;

public class Loadclasstest {
    public static void main(String[] args) throws Exception{
        ClassLoader cl = ClassLoader.getSystemClassLoader();
        Class<?> c = cl.loadClass("LoadClass.Person");
    }
}           //没有输出
```

`ClassLoader.loadClass()`不会进行类的初始化，如果用`newInstance()`实例化则会由输出。

## 动态加载字节码

### 字节码的概念

字节码是 java 编译器生成的中间代码，介于源码与机器码之间，是 JVM 能够理解的指令集，被存储在 .class 文件中。

经过 java 编译器编译出的 class 文件实际是一个二进制文件，但不是机器码，我们通过 idea 打开 class 文件看到的实际上是经过 idea 反编译的代码。

#### 类加载器的原理

调用 ClassLoader 的 loadClass() 方法时，会调用到内部的 findClass() 方法，而 ClassLoader 的 findClass 方法其实是没有实现的，所以到底会调用谁的 findClass 呢？由于 findClass() 是一个虚方法，运行时是根据对象的类型决定的，对象是一个自定义的对象，所以就是尝试调用`AppClassLoader`的 findClass 方法，但这个类没有重写这个方法，所以要去这个类的父类，即`URLClassLoader`来调用 findClass 方法。在 findClass 内部又调用了自身的 defineClass 方法，在自身的 defineClass 方法中又调用了父类的 defineClass 方法。所以，类加载的最终出口还是 ClassLoader 的 defineClass 方法。

由于类加载器直接加载字节码，如果我们在反序列化中能够控制类加载器加载远程字节码，就会有更多攻击手段。

### URLClassLoader 加载远程 class 文件

`URLClassLoader`实际上是我们默认使用的`AppClassLoader`的父类，在学习`URLClassLoader`的工作过程其实就是在学习默认类加载器的工作流程。

一般情况下，java 会根据配置项`sun.boot.class.path`和`java.class.path`中列举的基础路径（这些路径是经过处理的`java.net.URL`类）来寻找 .class 文件加载，这个路径分三种情况：

1. URL 没有以 / 结尾，则认为是一个 jar 包，使用`JarLoader`来寻找类，即在 jar 包中寻找 class 文件
2. URL 以 / 结尾，且协议名为 file，则使用`FileLoader`来寻找类，在本地文件系统寻找 class 文件
3. URL 以 / 结尾，协议名不是 file，则使用基本的`Loader`来寻找类

#### file 协议

在测试目录新建一个 Calc.java：

```java
import java.io.IOException;

public class Calc {
    static {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

在终端中 javac 编译一下，产生 class 文件。

测试代码样例：

```java
package LoadClass;

import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;

public class URLloadTest {
    public static void main(String[] args) throws MalformedURLException, ClassNotFoundException, InstantiationException, IllegalAccessException {
        URLClassLoader ucl = new URLClassLoader(new URL[]{
                new URL("file:///C:\\Users\\32202\\Desktop\\java测试\\out\\production\\test\\")});
        Class<?> calc = ucl.loadClass("Calc");
        calc.newInstance();
    }
}
```

ucl 是一个指向这个 Calc.class 的类加载器实例，调用 loadClass 方法来加载这个类再实例化就能弹出计算器。

#### HTTP 协议

在 Calc.class 所在目录下执行`python -m http.server 4444`建立 HTTP 服务，编写恶意类：

```java
package LoadClass;

import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;

public class URLloadTest {
    public static void main(String[] args) throws MalformedURLException, ClassNotFoundException, InstantiationException, IllegalAccessException {
        URLClassLoader ucl = new URLClassLoader(new URL[]{
                new URL("http://localhost:4444")});
        Class<?> calc = ucl.loadClass("Calc");
        calc.newInstance();
    }
}
```

也能弹出计算器。

#### file+jar 协议

先把这个 Calc.class 文件打包成 jar 包：`jar -cvf Calc.jar Calc.class`

调用恶意类：

```java
package LoadClass;

import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;

public class URLloadTest {
    public static void main(String[] args) throws MalformedURLException, ClassNotFoundException, InstantiationException, IllegalAccessException {
        URLClassLoader ucl = new URLClassLoader(new URL[]{
                new URL("jar:file:///C:\\Users\\32202\\Desktop\\java测试\\out\\production\\test\\Calc.jar!/")});
        Class<?> calc = ucl.loadClass("Calc");
        calc.newInstance();
    }
}
```

也能弹出计算器。

#### HTTP+jar 协议

```java
package LoadClass;

import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;

public class URLloadTest {
    public static void main(String[] args) throws MalformedURLException, ClassNotFoundException, InstantiationException, IllegalAccessException {
        URLClassLoader ucl = new URLClassLoader(new URL[]{
                new URL("jar:http://localhost:4444/Calc.jar!/")});
        Class<?> calc = ucl.loadClass("Calc");
        calc.newInstance();
    }
}
```

同样能弹出计算器。

对比之下其实还是 http 协议最灵活，能干更多的事情。

### ClassLoader#defineClass 直接加载字节码

在上面分析过，类加载的终点是 ClassLoader 的 defineClass 方法，它将传入的字节码转化成真正的 java 类。我们可以利用这个方法实现攻击。

ClassLoader#defineClass：

```java
protected final Class<?> defineClass(String name, byte[] b, int off, int len)
        throws ClassFormatError
    {
        return defineClass(name, b, off, len, null);
    }
```

`name`是类名，`b`是字节数组，`off`是偏移量，即从字节数组的生命位置开始读取，`len`是字节数组的长度。

这是一个受保护的方法，利用反射调用该方法实现字节码的加载，实例化后可进行其他攻击手法。

```java
package LoadClass;

import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.nio.file.Files;
import java.nio.file.Paths;

public class DefineClassTest {
    public static void main(String[] args) throws NoSuchMethodException, IOException, InvocationTargetException, IllegalAccessException, InstantiationException {
        ClassLoader scl = ClassLoader.getSystemClassLoader();
        Method dc = ClassLoader.class.getDeclaredMethod("defineClass",
                String.class, byte[].class, int.class, int.class);
        dc.setAccessible(true);
        byte[] code = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\java测试\\out\\production\\test\\Calc.class"));
        Class calc =(Class)dc.invoke(scl, "Calc", code, 0, code.length);
        calc.newInstance();
    }
}
```

### Unsafe 加载字节码

Unsafe 类中也有 defineClass 方法，也是 defineClass 加载字节码。

Unsafe 类里面有个属性`theUnsafe`，值是一个 Unsafe 对象，但是我们实际是无法直接拿到这个属性的。该类里面也有一个`getUnsafe`方法会返回这个`theUnsafe`属性，虽然该方法是 public 的，但是由于是采用单例模式设计的，也无法直接调用，所以需要反射进行调用。

```java
package LoadClass;

import sun.misc.Unsafe;

import java.io.IOException;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.security.ProtectionDomain;

public class UnsafeLoadTest {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, NoSuchMethodException, IOException, InvocationTargetException, InstantiationException {
        ClassLoader cl = ClassLoader.getSystemClassLoader();
        Class<Unsafe> unsafeClass = Unsafe.class;
        Field theUnsafe = unsafeClass.getDeclaredField("theUnsafe");
        theUnsafe.setAccessible(true);
        Unsafe o = (Unsafe) theUnsafe.get(null);
        Method defineClass = unsafeClass.getDeclaredMethod("defineClass", String.class, byte[].class,
                int.class, int.class,ClassLoader.class, ProtectionDomain.class);
        byte[] code = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\java测试\\out\\production\\test\\Calc.class"));
        Class calc = (Class)defineClass.invoke(o, "Calc", code, 0, code.length, cl, null);
        calc.newInstance();
    }
}
```

### TemplatesImpl 加载字节码

定位到`TemplatesImpl`类，我们会找到一个内部类`TransletClassLoader`，该类继承了`ClassLoader`，并且重写了`defineClass`方法。

源码：

```java
static final class TransletClassLoader extends ClassLoader {
        private final Map<String,Class> _loadedExternalExtensionFunctions;

         TransletClassLoader(ClassLoader parent) {
             super(parent);
            _loadedExternalExtensionFunctions = null;
        }

        TransletClassLoader(ClassLoader parent,Map<String, Class> mapEF) {
            super(parent);
            _loadedExternalExtensionFunctions = mapEF;
        }

        public Class<?> loadClass(String name) throws ClassNotFoundException {
            Class<?> ret = null;
            // The _loadedExternalExtensionFunctions will be empty when the
            // SecurityManager is not set and the FSP is turned off
            if (_loadedExternalExtensionFunctions != null) {
                ret = _loadedExternalExtensionFunctions.get(name);
            }
            if (ret == null) {
                ret = super.loadClass(name);
            }
            return ret;
         }

        /**
         * Access to final protected superclass member from outer class.
         */
        Class defineClass(final byte[] b) {
            return defineClass(null, b, 0, b.length);
        }
    }
```

这里我们可以看到，defineClass 方法的访问权限从父类的受保护类型变成了现在的默认类型，这样一来，在同一个包内的`TemplatesImpl`就能直接调用它来加载字节码。

从`TransletClassLoader#defineClass()`追溯调用链：

```text
TemplatesImpl#getOutputProperties() -> TemplatesImpl#newTransformer() ->

TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses()

-> TransletClassLoader#defineClass()
```

前两个入口的源码：

```java
public synchronized Transformer newTransformer()
        throws TransformerConfigurationException
    {
        TransformerImpl transformer;

        transformer = new TransformerImpl(getTransletInstance(), _outputProperties,
            _indentNumber, _tfactory);

        if (_uriResolver != null) {
            transformer.setURIResolver(_uriResolver);
        }

        if (_tfactory.getFeature(XMLConstants.FEATURE_SECURE_PROCESSING)) {
            transformer.setSecureProcessing(true);
        }
        return transformer;
    }

    /**
     * Implements JAXP's Templates.getOutputProperties(). We need to
     * instanciate a translet to get the output settings, so
     * we might as well just instanciate a Transformer and use its
     * implementation of this method.
     */
    public synchronized Properties getOutputProperties() {
        try {
            return newTransformer().getOutputProperties();
        }
        catch (TransformerConfigurationException e) {
            return null;
        }
    }
```

在`newTransformer()`内部直接调用`newTransformer()`，在这个方法内，`transformer = new TransformerImpl(getTransletInstance(), _outputProperties, _indentNumber, _tfactory);`，这里很明显调用了`getTransletInstance()`方法，这就意味着每次调用`newTransformer()`来创建`TransformerImpl`对象时都会去执行`getTransletInstance()`。

```java
private Translet getTransletInstance()
        throws TransformerConfigurationException {
        try {
            if (_name == null) return null;

            if (_class == null) defineTransletClasses();

            // The translet needs to keep a reference to all its auxiliary
            // class to prevent the GC from collecting them
            AbstractTranslet translet = (AbstractTranslet) _class[_transletIndex].newInstance();
            translet.postInitialization();
            translet.setTemplates(this);
            translet.setServicesMechnism(_useServicesMechanism);
            translet.setAllowedProtocols(_accessExternalStylesheet);
            if (_auxClasses != null) {
                translet.setAuxiliaryClasses(_auxClasses);
            }

            return translet;
        }
        catch (InstantiationException e) {
            ErrorMsg err = new ErrorMsg(ErrorMsg.TRANSLET_OBJECT_ERR, _name);
            throw new TransformerConfigurationException(err.toString());
        }
        catch (IllegalAccessException e) {
            ErrorMsg err = new ErrorMsg(ErrorMsg.TRANSLET_OBJECT_ERR, _name);
            throw new TransformerConfigurationException(err.toString());
        }
    }
```

第一步安检：`if (_name == null) return null;`，这表明在利用时必须要利用反射给这个属性赋任意值，不然调用链直接断了。

第二步安检：`if (_class == null) defineTransletClasses();`，`_class`为空时才调用`defineTransletClasses()`把字节码转化为`Class`对象。

注：`_class`是一个`Class[]`数组，用来存放已经加载过的类，防止重复加载。对于一个全新的`TemplatesImpl`实例，`_class`默认是`null`，所以首次调用一定会调用`defineTransletClasses()`。

最终执行：`AbstractTranslet translet=(AbstractTranslet)_class[_transletIndex].newInstance();`

1. `_class[_transletIndex]`取出恶意类。`_class`数组已经在`defineTransletClasses()`赋值。`_transletIndex`是在那个方法中确定的，指向是那个父类为 `AbstractTranslet` 的类在数组中的位置
2. `.newInstance()`实例化，这里就是触发点，如果恶意代码写在类的静态代码块或者无参构造方法中则会执行。

```java
private void defineTransletClasses()
        throws TransformerConfigurationException {

        if (_bytecodes == null) {
            ErrorMsg err = new ErrorMsg(ErrorMsg.NO_TRANSLET_CLASS_ERR);
            throw new TransformerConfigurationException(err.toString());
        }

        TransletClassLoader loader = (TransletClassLoader)
            AccessController.doPrivileged(new PrivilegedAction() {
                public Object run() {
                    return new TransletClassLoader(ObjectFactory.findClassLoader(),_tfactory.getExternalExtensionsMap());
                }
            });

        try {
            final int classCount = _bytecodes.length;
            _class = new Class[classCount];

            if (classCount > 1) {
                _auxClasses = new HashMap<>();
            }

            for (int i = 0; i < classCount; i++) {
                _class[i] = loader.defineClass(_bytecodes[i]);
                final Class superClass = _class[i].getSuperclass();

                // Check if this is the main class
                if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
                    _transletIndex = i;
                }
                else {
                    _auxClasses.put(_class[i].getName(), _class[i]);
                }
            }

            if (_transletIndex < 0) {
                ErrorMsg err= new ErrorMsg(ErrorMsg.NO_MAIN_TRANSLET_ERR, _name);
                throw new TransformerConfigurationException(err.toString());
            }
        }
        catch (ClassFormatError e) {
            ErrorMsg err = new ErrorMsg(ErrorMsg.TRANSLET_CLASS_ERR, _name);
            throw new TransformerConfigurationException(err.toString());
        }
        catch (LinkageError e) {
            ErrorMsg err = new ErrorMsg(ErrorMsg.TRANSLET_OBJECT_ERR, _name);
            throw new TransformerConfigurationException(err.toString());
        }
    }
```

这是利用链中最核心的部分，负责把`_bytecodes`里的字节码变成可执行的`Class`对象，并筛选出恶意类。

```java
TransletClassLoader loader = (TransletClassLoader)
    AccessController.doPrivileged(new PrivilegedAction() {
        public Object run() {
            return new TransletClassLoader(ObjectFactory.findClassLoader(),
                _tfactory.getExternalExtensionsMap());
        }
    });
```

这里创建了一个自定义类加载器，这是上面所讲的`TransletClassLoader`这个内部类，继承于`ClassLoader`。然后则是遍历并加载所有字节码。

然后就是最关键的一步：父类检查。

```java
final Class superClass = _class[i].getSuperclass();

// Check if this is the main class
if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
    _transletIndex = i;
}
else {
    _auxClasses.put(_class[i].getName(), _class[i]);
}
```

这里判断了刚刚加载的类是不是`ABSTRACT_TRANSLET`，即`com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet`。

如果是，则把当前索引赋值给`_transletIndex`，标记这个类是主类。如果不是，则当作是辅助类，存入`_auxClasses`备用。

这就说明，我们的恶意类必须继承`AbstractTranslet`，否则最终无法被实例化。

到此调用链基本结束。

#### 利用 TemplatesImpl#newTransformer() 构造简单POC

根据上面的分析，我们构造的字节码必须继承于`AbstractTranslet`这个抽象类，所以需要重写一下里面的方法。

```java
package LoadClass;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class TemplatesBytes extends AbstractTranslet {

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}

    public TemplatesBytes() throws Exception{
        super();
        Runtime.getRuntime().exec("Calc");
    }
}
```

```java
package LoadClass;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.io.File;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;

public class TemplatesImpILoadTest {

    public static void setFieldval(Object obj,String fieldName,Object value) throws Exception{
        Field f = obj.getClass().getDeclaredField(fieldName);
        f.setAccessible(true);
        f.set(obj,value);
    }

    public static void main(String[] args) throws Exception{
        byte[] codes = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\java测试\\test\\src\\LoadClass\\TemplatesBytes.class"));
        TemplatesImpl templates = new TemplatesImpl();
        setFieldval(templates,"_name","Calc");
        setFieldval(templates,"_bytecodes",new byte[][]{codes});
        setFieldval(templates,"_tfactory",new TransformerFactoryImpl());
        /*这一步很重要，因为在源码中有
    return new TransletClassLoader(ObjectFactory.findClassLoader(),_tfactory.getExternalExtensionsMap());，而_tfactory默认是null,
    需要赋一个值来让程序走下去*/
        templates.newTransformer();
    }
}
```

这里在运行时会报一个空指针的错，但是没什么影响，依然会弹出计算器。

## 总结

类加载这部分内容感觉还是挺重要的，所以专门写了这一篇博客来记录一下我的学习成果，不过主要还是参考了别的大佬的内容，可以说内容几乎是一样的，但是其中也加了一些我自己的理解。

这一篇博客还只是对类加载的基本认识，中间有些地方还是有一点模糊的，但是相比之前已经有了很大改善了，也是有了不小的收获吧。之后还会有更多关于动态加载字节码的内容，遇到之后会继续更新。

























