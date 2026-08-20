---
title: JS应用
date: 2026-08-04
---

# JS 应用

## Ajax 技术

Ajax 全称异步 Javascript 和 XML，这一项技术主要是用来发送 HTTP 请求的。

Ajax 允许通过与场景后面的 Web 服务器交换数据来异步更新网页。这意味着可以更新网页的部分，而不需要重新加载整个页面。

### Ajax 如何工作

![ajax工作流程图](https://www.w3school.com.cn/i/ajax.gif)

1. 网页中发生一个事件（页面加载、按钮点击）
2. 由 JavaScript 创建 XMLHttpRequest 对象
3. XMLHttpRequest 对象向 web 服务器发送请求
4. 服务器处理该请求
5. 服务器将响应发送回网页
6. 由 JavaScript 读取响应
7. 由 JavaScript 执行正确的动作（比如更新页面）

### 3种 Ajax 技术

1. 原生 Ajax

```js
var xhttp = new XMLHttpRequest();
	xhttp.open("GET", "./1.txt", true);
	xhttp.send();
	xhttp.onreadystatechange = function() {
		if (this.readyState == 4 && this.status == 200) {
		document.getElementById("content").innerHTML = this.responseText;
	}
};
```

2. jQuery 库

```js
$.ajax({
	method: "GET",
	url: "1.txt",
	dataType: "text",
	success: function(data) {
	document.write(data);
	}
});
```

3. Axios 库

```js
axios({
	method:'GET',
	url:'1.txt,
	dataType:'text',
}).then(function(response){
		console.log(response.data);
	})
//或者下面这种更简单的写法
axios.get('1.txt').then(function(response){
     console.log(response.data);
})
```

目前主流的 Ajax 技术是 jQuery 库和 Axios 库。

## BOM 浏览器对象

DOM 其实属于 BOM 的一种。

### Window 对象

所有浏览器都支持 **window** 对象。它代表浏览器的窗口或者标签页。

所有全局 JavaScript 对象，函数和变量自动成为 window 对象的成员。

全局变量是 window 对象的属性。

全局函数是 window 对象的方法。

甚至（HTML DOM 的）document 对象也是 window 对象属性：

```js
window.document.getElementById("header");
```

等同于：

```js
document.getElementById("header");
```

### Screen 对象

这个对象存储的是用户显示器的硬件和系统配置信息

**window.screen** 对象不带 window 前缀也可以写：

### 属性：

- `screen.width`
- `screen.height`
- `screen.availWidth`
- `screen.availHeight`
- `screen.colorDepth`
- `screen.pixelDepth`

有些页面为了适配各种设备，会用这个对象进行对应的调整。

### Location 对象

这个对象存储的是当前网页地址的所有信息。

**window.location** 对象可不带 window 前缀书写。

一些例子：

- `window.location.href` 返回当前页面的 href (URL)
- `window.location.hostname` 返回 web 主机的域名
- `window.location.pathname` 返回当前页面的路径或文件名
- `window.location.protocol` 返回使用的 web 协议（http: 或 https:）
- `window.location.assign` 加载新文档

### Navigator 对象

这个对象存储浏览器本身的软硬件环境信息。

**window.navigator** 对象可以不带 window 前缀来写。

一些例子：

- `navigator.appName`
- `navigator.appCodeName`
- `navigator.platform`
- `navigator.userAgent`

### History 对象

这个对象存储的是浏览器的历史信息。

**window.history** 对象可不带 window 书写。

为了保护用户的隐私，JavaScript 访问此对象存在限制。

### 一些方法：

- `history.back()` - 等同于在浏览器点击后退按钮
- `history.forward()` - 等同于在浏览器中点击前进按钮

## DOM 文档树

当网页被加载时，浏览器会创建页面的文档对象模型。

HTML DOM 模型会被结构化为对象树：

![对象的HTML DOM树](https://www.w3school.com.cn/i/ct_htmltree.gif)

通过这个对象模型，js 能够创建动态 HTML：

- JavaScript 能改变页面中的所有 HTML 元素
- JavaScript 能改变页面中的所有 HTML 属性
- JavaScript 能改变页面中的所有 CSS 样式
- JavaScript 能删除已有的 HTML 元素和属性
- JavaScript 能添加新的 HTML 元素和属性
- JavaScript 能对页面中所有已有的 HTML 事件作出反应
- JavaScript 能在页面中创建新的 HTML 事件

样例代码：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>

<a id="a" href="http://www.baidu.com">点击</a>

<script >
    let a = document.getElementById('a').href;
    a.href = 'http://www.youku.com';
</script>

<!--<button onclick="func()">点击</button>-->
<!--<script>-->
<!--    function func() {-->
<!--        alert('1');-->
<!--    }-->
<!--</script>-->

<button id="bn">点击</button>
<script>
    let bn = document.getElementById('bn');
    bn.onclick = function () {alert('xss');}
</script>
</body>
</html>
```

比如我们点击搜索框出现之前的搜索记录就和 DOM 相关。

## 前端加密

### 数据加密

#### crypto-js库

样例代码：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>

<script src="crypto-js.js"></script>

<script>

    var str='solowalker';

    //base64编码
    var base64str=window.btoa(str);
    console.log(base64str);
    //md5加密
    var md5str=CryptoJS.MD5(str).toString();
    console.log(md5str);
    //SHA1加密
    var sha1str = CryptoJS.SHA1(str).toString();
    console.log(sha1str);
    //HMAC加密
    var key='key';
    var hash = CryptoJS.HmacSHA256(str,key);
    var HMACstr=CryptoJS.enc.Hex.stringify(hash);
    console.log(HMACstr);
    //AES加密
    var aeskey='aeskey';
    var aesstr=CryptoJS.AES.encrypt(str,CryptoJS.enc.Utf8.parse(aeskey),
        {
            mode:CryptoJS.mode.ECB,
            padding: CryptoJS.pad.Pkcs7
        }
    ).toString();
    console.log(aesstr);
    //DES加密
    var deskey='deskey';
    var desstr=CryptoJS.DES.encrypt(str,CryptoJS.enc.Utf8.parse(deskey),
        {
            mode:CryptoJS.mode.ECB,
            padding: CryptoJS.pad.Pkcs7
        }
    ).toString();
    console.log(desstr);

</script>

</body>
</html>
```

这个库并不支持非对称加密，所以进行非对称加密时需要使用另一个库——JSEncrypt。

#### JSEncrypt 库

样例代码：

```js
var PUBLIC_KEY='-----BEGIN PUBLIC KEY-----\n' +
	'MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAx4RhaeXjBM+2CrWPNPXc\n' +
	'1NKN1+kKHHdBL5rMDzH6RAAx3KClHSjavq/vvWRDbyIOGWIaQdyiq5KBXlvJ/B7J\n' +
	'vnvrduGY7wCjUb6J5RNLiUxAvyX2wm6QHaRjy5jJjCEr2ZL4F9tzdFX5Bfl+vxHs\n' +
	'3cBlvf0d39ZpQNA+SEoP5UK51hUWtnM3m/lkzSvW4zZ8fV4GrbNOnP7fYeK4SltX\n' +
	'KQQjYCbgxX5erwn4NeQRfr6VIk3tT66Jr1qM/uQwDzPhDmBI3UlebMJsCtXHvIVs\n' +
	'mObJInFTwlT9MoEDyoGAhD6AiPhUjVQMG2rI/DlZ7q18WQIIGO3Hmo28b8PTeO5v\n' +
	'MQIDAQAB\n' +
	'-----END PUBLIC KEY-----\n'

var PRIVATE_KEY='-----BEGIN PRIVATE KEY-----\n' +
	'MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDHhGFp5eMEz7YK\n' +
	'tY809dzU0o3X6Qocd0EvmswPMfpEADHcoKUdKNq+r++9ZENvIg4ZYhpB3KKrkoFe\n' +
	'W8n8Hsm+e+t24ZjvAKNRvonlE0uJTEC/JfbCbpAdpGPLmMmMISvZkvgX23N0VfkF\n' +
	'+X6/EezdwGW9/R3f1mlA0D5ISg/lQrnWFRa2czeb+WTNK9bjNnx9Xgats06c/t9h\n' +
	'4rhKW1cpBCNgJuDFfl6vCfg15BF+vpUiTe1PromvWoz+5DAPM+EOYEjdSV5swmwK\n' +
	'1ce8hWyY5skicVPCVP0ygQPKgYCEPoCI+FSNVAwbasj8OVnurXxZAggY7ceajbxv\n' +
	'w9N47m8xAgMBAAECggEBAJLKyUk6xE6T7CHw9w5GHlXPHGFQqgnLjABUafJ8GN/T\n' +
	'LNxgrVrI5jgKBd9YV2z6p1jxntP6WwzU263q5q9Cj7hAQDvVO8oMtBy+jYInMhow\n' +
	'Kir46Zaf9hR5EJuJLDCXb0XDJhmXcliTsIY+zIwTyixVFAY2prY7cHEpRcm2A//u\n' +
	'TWOr2Q9NuPyKhuoSTkVN2uJqKIWTOFt7rYVicHzWr9X2ImdHdUoxXpsrwJHulrGO\n' +
	'p3J3Nr46VyTjsgk15KQWqfeo6skWfPps9uiG7dHG+Jt35yfHkEBvEkSIBw1MXuRc\n' +
	'oh9l1v7Y8mj1uK2B64CfDnifig+VPcg70qpjQr3ATGECgYEA5goH+vPzIzDdy4pS\n' +
	'UTE2eWcdfH+0VRinh9MxI/mREichTr4FWiSfPhwecay/i2ccadNIiiokfBWIwfyU\n' +
	'42ZNV5QUP83ZJcAXSUgQ1haJ1yRgHEN7kb2Q1y0H34m5/rCSVGUzLB3AQ5ptNtU6\n' +
	'JdyvAX2Bqm6aBuICNBjlo23QRr0CgYEA3giMa283OZ+++hZnsNRZnAXcwNhGUgHf\n' +
	'SNsX89JZoCRainzkj/l35rCeLqNPnDBC15OJ0GcaNmE4AE8Ms410n0EiHIYdKeDO\n' +
	'K2gsc17AAybnDjj1RO7y60WLFsglco27PMdWOZorC/7D53GZUGh1Af1KDIYcgijp\n' +
	'V3uWPeJu24UCgYBVJ3l0yEFE0Z4I7pcyPwlvP2CG6a8ToSDDAsa6DnRJR/robycE\n' +
	'C3J3R2ltowj0zaKS+gdsPdVrqX0KcjmbRA91T/d+9vBfLRBxrB+vYIB+B5UcYU6o\n' +
	'0IeBX8X+VbloMmy4mQ2sUwcM/2lWVvBDe8G8x3zsXizeR2ORbXX0XX4v8QKBgFn/\n' +
	'FQum9LeCrKIp2rWuHPRE3Am+oCI1aA/b3oWRyYpDsf9YSDyjXZpAFJ3KzEX+udkv\n' +
	'kDjM0a8hENXvNLLCr3atq+nr4n5LBMZLX1kUGrgsWJNHOwNJ52S9t3bwgV1BXZdx\n' +
	'JN4MQ06FYVq6jO4uqN65j/4rjfqkIpC3I1rKIS0RAoGAJeeonwQmcXFPvIjMtpJl\n' +
	'yHcRKWvCyIR2IAg/8pBEsAa2pknFMrR5psJId57F15ND5jf1MAtHUBdmTQ2A04y6\n' +
	'yvMHacFgBpsLH1rE5meCRUGOUN00z6cwbZsRulinDd9YuNTkalJ6dxZfsvEx5sJ+\n' +
	'akHZ6hneq3bbvlFqcC8gB4g=\n' +
	'-----END PRIVATE KEY-----\n'

//公钥加密
var encrypt = new JSEncrypt();//实例化加密对象
encrypt.setPublicKey(PUBLIC_KEY);//设置公钥
var encrypted = encrypt.encrypt(str);//对指定数据进行加密
console.log(encrypted);

//私钥解密
var decrypt = new JSEncrypt(); // 创建解密对象
decrypt.setPrivateKey(PRIVATE_KEY); //设置私钥
var uncrypted = decrypt.decrypt(encrypted); //解密
console.log(uncrypted);
```

### 代码混淆

混淆代码的主要目的是保护源代码，防止未经授权的复制、篡改或逆向工程。通过对变量名、字符串和控制流的修改，混淆代码看似毫无逻辑，但本质功能没变。混淆技术常用于商业软件和恶意软件中。

网上也有很多在线 js 混淆的网站，更多技术会在后续 js 逆向学习到。

## Node.js

Node.js 是让 javascript 运行在服务端的运行环境。

样例代码：

```js
const fs = require('fs');
const express = require('express');
const app = express();

app.get('/', (req, res) => {

    var name = req.query.name;
    res.send(name);

    fs.readFile('1.txt', 'utf8', ( err, data) => {
        if (err) {throw err;}
        //res.send(data);
    })
})


var server = app.listen(8081, '0.0.0.0',function() {
    var port = server.address().port;
    var host = server.address().address;
    console.log(host);
});
```

```js
const child_Process = require('child_process');

//child_Process.exec('calc');

child_Process.spawnSync('calc');
```

Node.js 的相关漏洞以原型链污染等为特色。

## Webpack

Webpack 是一个强大的模块打包工具，主要用于将 Javascript 代码和其他资源（如 CSS、图片、字体等）打包成浏览器能够高效加载的文件。

Webpack支持模块化开发，可以将代码分割成多个文件（模块），然后将这些文件按需打包成一个或多个最终的输出文件。这对于管理复杂应用程序的代码非常重要，特别是现代 JavaScript 应用程序中，大多数代码和资源都已是模块化的。例如，你可以将前端代码分为多个模块（如组件、工具函数等），然后让 Webpack 负责打包这些模块，并管理它们之间的依赖关系。

Webpack 不仅仅是处理 JavaScript 文件，还能处理多种类型的资源：

- CSS：你可以使用 Webpack 将 CSS 文件打包到最终的输出中。

- Sass / Less：Webpack 配合相应的加载器（如 sass-loader、less-loader）可以处理 Sass 或 Less 文件。

- 图片和字体：通过 file-loader 或 url-loader，Webpack 可以处理图片、字体文件等静态资源。

- HTML 文件：可以使用 html-webpack-plugin 插件自动生成 HTML 文件并插入打包后的资源。

Webpack 支持 代码分割，它可以将大型的 JavaScript 应用程序拆分成多个小的文件（chunks），并按需加载这些文件。这样，初次加载时，浏览器只会加载最小的必需代码，而不是所有的代码，从而提高页面加载速度。例如，你可以按页面或功能进行代码分割，只加载用户需要的部分，而不是一次性加载整个应用程序。

### 源码泄露

其中一种是由打包模式引起的源码泄露。

如果打包模式是开发模式，我们就可以在开发者工具中看到源码。还原：浏览器 Webpack。

第一种就是 devtool 配置不当，将会在部署代码文件中生成对应匹配的 sourcemap 文件（源码映射），如果将参数 devtool 配置为“source-map”、“cheap-source-map”、“hidden-source-map”、“nosources-source-map”、“cheap-module-source-map”等值是，打包会生成单独的 map 文件。

## Vue

**Vue.js**（简称 **Vue**）是一个用于构建用户界面的**渐进式 JavaScript 框架**。它由尤雨溪（Evan You）于 2014 年创建，目前是全球最流行的前端框架之一。

Vue 项目可以由 Vite 进行构建，有开发模式和生产模式。

| 维度              | 开发模式 (Development)           | 生产模式 (Production)              |
| :---------------- | :------------------------------- | :--------------------------------- |
| **启动方式**      | `npm run dev`                    | `npm run build`                    |
| **目的**          | 方便你写代码、调试               | 给用户访问的最终版本               |
| **代码体积**      | 大（包含注释、空格、完整变量名） | 小（压缩、混淆、tree-shaking）     |
| **警告/错误提示** | ✅ 详细（有完整堆栈和友好提示）   | ❌ 精简（只显示必要错误）           |
| **性能**          | 较慢（按需编译、实时转译）       | 极快（预先优化、缓存）             |
| **热更新 (HMR)**  | ✅ 支持（改代码立即看到效果）     | ❌ 不支持（是静态文件）             |
| **调试工具**      | ✅ 支持 Vue Devtools 完整功能     | ⚠️ 部分功能受限                     |
| **Source Map**    | ✅ 有（方便定位错误行号）         | ⚠️ 可选（默认关闭以保护源码）       |
| **构建产物**      | 不生成文件（直接在内存中运行）   | 生成 `dist/` 文件夹（HTML/CSS/JS） |
| **部署方式**      | 本地服务器（只给自己看）         | 部署到线上服务器（给所有用户看）   |

Vue 这个前端框架非常安全，很少会出现漏洞。

如果出现 v-html 这个属性会导致 XSS 漏洞，目前 Vue 推荐使用 `{{}}`来处理变量，避免了 XSS 漏洞。

Vue 加上 Webpack 一样可能会导致类似的源码泄露问题。

## 小程序

这里小程序以微信小程序为主。

### 小程序架构

1. 主体结构 

小程序包含一个描述整体结构 app 和多个描述各自页面的 page。一个小程序主体部分（即 app）由三个文件组成，必须放在项目的根目录：

| 文件     | 必需 | 作用             |
| -------- | ---- | ---------------- |
| app.js   | 是   | 小程序逻辑       |
| app.json | 是   | 小程序公共配置   |
| app.wxss | 否   | 小程序公共样式表 |

2. 一个小程序页面由四个文件组成：

| xxx.js   | 页面逻辑 |
| -------- | -------- |
| xxx.json | 页面配置 |
| xxx.wxml | 页面结构 |
| xx.wxss  | 页面样式 |

3. 页面整体目录结构

| pages               | 页面文件夹                           |
| ------------------- | ------------------------------------ |
| index               | 首页                                 |
| logs                | 日志                                 |
| utils               | 存放工具类文件夹                     |
| app.js              | 入口 js                              |
| app.json            | 全局配置文件                         |
| app.wxss            | 全局样式文件                         |
| project.config.json | 配置                                 |
| sitemap.json        | 配置小程序及其页面是否允许被微信索引 |

### 安全问题

1. 逆向反编译
2. 敏感信息泄露（AK/SK，APIKEY等）
3. 资产信息提取（IP、web资产）
4. 代码逻辑安全（算法、提交接口等）



















































