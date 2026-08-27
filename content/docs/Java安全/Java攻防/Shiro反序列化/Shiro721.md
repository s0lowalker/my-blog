---
title: Shiro721
date: 2026-08-25
---

# Shiro721

## 前言

在 Shiro550 漏洞中，Cookie 所使用的 AES 加密密钥为硬编码，所以我们可以构造恶意序列化数据并使用固定的 AES 密钥进行正确加密恶意序列发送给服务端，进而达到攻击的目的。但在该漏洞公布后，Shiro 官方修复了这一漏洞，将AES密钥修改成了动态生成。也就是说，对于每一个 Cookie，都是使用不同的密钥进行加解密的。

## 环境配置

这个环境配置感觉还是有点麻烦的。

[SHIRO-721/samples-web-1.4.1.war at master · jas502n/SHIRO-721](https://github.com/jas502n/SHIRO-721/blob/master/samples-web-1.4.1.war)先安装这个样例 war 包，然后解压，把`WEB-INF/lib`的`CommonsCollections`包换成3.2.1版本的。然后把解压后的文件夹放到 tomcat 的 webapps 目录中，在 idea 中配置一下 URL 之后再启动。

解压后的文件夹也可以在项目的 webapp 目录复制一份，在项目结构的设置里找到库设置，在里面添加上解压后的文件夹的`WEB-INF/lib`。

## 漏洞复现

### 前提条件

漏洞影响的版本是 1.2.5<=shiro<=1.4.1。

Apache Shiro Padding Oracle Attack 的漏洞利用必须满足如下前提条件：

- 开启 rememberMe 功能
- rememberMe 值使用 AEC-CBC 模式解密
- 能获取正常 Cookie，即用户正常登录的 Cookie 值
- 密文可控

### 复现

先正常登录，勾选 rememberMe 选项：

![1](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/login.png)

点击 login 之后就能获取此时登录的 rememberMe 值：

![2](/images/screenshots/屏幕截图 2026-08-25 215154.png)

然后用 yso 生成 URLDNS 来验证：

```java
java -jar ysoserial-all.jar URLDNS "http://jrvghlhpwb.yutu.eu.org" > urldns.class
```

再利用 github 上的 exp [github.com/inspiringz/Shiro-721](https://github.com/inspiringz/Shiro-721)进行 Padding Oracle Attack，获取恶意 Cookie。再把生成的 Cookie 替换掉，发包之后成功出现 DNSlog。

## 漏洞分析

### Padding Oracle Attack 构造加密数据分析

简单说一下 Padding Oracle Attack 加密数据的整体过程：

1. 选择一个明文`P`，用来生成你想要的密文`C`
2. 使用适当的 Padding 将字符串填充为块大小的倍数，然后将其拆分成从 1 到 N 的块
3. 生成一个随机数据块（C[n] 表示最后一个密文块）
4. 对于每一个明文块，从最后一块开始：
   - 创建一个包括两块的密文 C‘，其实是通过一个空块（00000...）与最近生成的密文块 C[n+1]（如果是第一轮则是随机块）组合而成
   - 这一步比较容易理解，就是 Padding Oracle 的基本攻击原理：修改空块的最后一个字节直至 Padding Oracle 没有出现错误为止，然后继续将最后一个字节设置为2并修改最后第二个字节直至 Padding Oracle 没有出现错误为止，依次类推，直至最后一个数据为止
   - 在计算完整个块之后，将它与明文块 P[n] 进行XOR一起创建 C[n]
   - 对后续的每个块重复上述的过程（在新的密文块前添加一个空块，然后进行 Padding Oracle 爆破计算）

简单地说，每一个密文块解密为一个未知值，然后与前一个密文块进行XOR。通过仔细选择前一个块，我们可以控制下一个块解密来得到什么。即使下一个块解密为一堆无用数据，但仍然能被XOR化为我们控制的值，因此可以设置为任何我们想要的值。

### 漏洞代码分析

#### 密钥生成

在 shiro550 中密钥是硬编码的，而在 shiro721 中，密钥的生成方式变为动态生成：

```java
public AbstractRememberMeManager() {
    AesCipherService cipherService = new AesCipherService();
    this.cipherService = cipherService;
    this.setCipherKey(cipherService.generateNewKey().getEncoded());
}
```

shiro 通过`generateNewKey()`方法获取密钥：

```java
public Key generateNewKey() {
    return this.generateNewKey(this.getKeySize());
}

public Key generateNewKey(int keyBitSize) {
    KeyGenerator kg;
    try {
        kg = KeyGenerator.getInstance(this.getAlgorithmName());
    } catch (NoSuchAlgorithmException e) {
        String msg = "Unable to acquire " + this.getAlgorithmName() + " algorithm.  This is required to function.";
        throw new IllegalStateException(msg, e);
    }

    kg.init(keyBitSize);
    return kg.generateKey();
}
```

这里获取了一个密钥生成器，跟进到`init()`中：

![2](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/init.png)

![3](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/engineInit.png)

到`var4`这里是`AESKeyGenerator`，跟进`engineInit()`，进行 AES 算法的初始化。

![4](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/engineInitAG.png)

回到`org.apache.shiro.crypto.AbstractSymmetricCipherService#generateNewKey()`，跟进到`generateKey()`。

![5](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/generateKey.png)

在`com.sun.crypto.provider.AESKeyGenerator#engineGenerateKey()`下一个断点：

![6](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/engineGenerateKey.png)

这里生成了一串16字节的随机序列，并且返回一个`SecretKeySpec`对象，再使用`getEncoded()`方法获取`key`密钥序列。

![7](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/SecretKeySpec.png)

这就是密钥的完整生成过程。

#### Padding Oracle Attack

要成功进行 Padding Oracle Attack 需要服务端返回两个不同的响应特征来进行布尔判断。

在 Apache Shiro 的场景中，这个服务端的两个不同的响应特征为：

- Padding Oracle 错误时，服务端响应报文的 Set-Cookie 头字段返回 `rememberMe=deleteMe`
- Padding Oracle 正确时，服务端返回正常的响应报文内容

我们可以通过响应头来判断明文填充是否正确，从而爆破出中间值。而对于解密不正确的 Cookie，shiro 也有特别的处理方法。

##### Padding 错误处理

解密函数在 `org.apache.shiro.mgt.AbstractRememberMeManager#decrypt()` 中：

![8](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/decrypt.png)

跟进到`cipherService.decrypt()`中，最后发现再`crypt()`中调用了`doFinal()`方法：

![9](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/crypt.png)

![10](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/doFinal.png)

```java
public final byte[] doFinal(byte[] var1) throws IllegalBlockSizeException, BadPaddingException {
    this.checkCipherState();
    if (var1 == null) {
        throw new IllegalArgumentException("Null input buffer");
    } else {
        this.chooseFirstProvider();
        return this.spi.engineDoFinal(var1, 0, var1.length);
    }
}
```

`doFinal()` 方法有 `IllegalBlockSizeException`和`BadPaddingException` 这两个异常，分别用于捕获块大小异常和填充错误异常。异常会被抛出到 `crypt()` 方法中，最终被 `getRememberedPrincipals()` 方法捕获，并执行 `onRememberedPrincipalFailure()` 方法。

![12](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/onRememberedPrincipalFailure.png)

`onRememberedPrincipalFailure()` 方法调用了 `forgetIdentity()`。该方法会调用 `removeFrom()`，并且会在 response 头部添加字段 `Set-Cookie: rememberMe=deleteMe`。

![13](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/forgetIdentity.png)

所以如果 Padding 结果不正确的话，响应包就会返回`Set-Cookie: rememberMe=deleteMe`。

##### Padding 正确，进行反序列化

CBC模式下的分组密码，如果某一组的密文被破坏，那么在其之后的分组都会受到影响。这时候我们的密文就无法正确的被反序列化了。

 Shiro 中关于反序列化的处理在`org.apache.shiro.io.DefaultSerializer#deserialize()`方法下：

![14](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/deserialize.png)

如果反序列化的结果错误，则会抛出异常，最后异常仍然会被`getRememberedPrincipals()`处理，response 包里会回显 302 且 `rememberMe=deleteMe`。

但是对于 Java 来说，反序列化是以流的方式按照顺序进行的，在后面添加或更改一些字符并不会影响正常反序列化。

我们获取正常用户的 Cookie 并使用密钥解密，可以看到最后填充的数据为`0x0B`：

![15](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/addTails.png)

下面改成其他合法填充方式，然后加密发送出去：

![16](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/changingPaddings.png)

![17](https://drun1baby.top/2023/03/08/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Shiro%E7%AF%8702-Shiro721%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/SendDecrypt.png)

服务器端正常响应，于是这里就构造出布尔条件：

- Padding 正确，服务器正常响应
- Padding 错误，服务器返回 `Set-Cookie: rememberMe=deleteMe`

## 总结

这个漏洞似乎并不是很好利用，但是好像在面试中会经常问到，所以还是要好好学习一下，而且这个漏洞也是扩展了一条攻击 CBC 的路线。

















