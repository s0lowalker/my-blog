---
title: node.js原型链污染和沙箱逃逸漏洞学习笔记
date: 2026-03-01
tags: [Node.js专题]
categories: 
  - web安全
  - Node.js安全
excerpt: node.js两个漏洞的基础学习
---

# node.js原型链污染和沙箱逃逸

## node.js原型链

#### 什么是原型（Prototype）

在 JavaScript 中，几乎所有对象都是通过原型关联的。每个对象（除了 null）都有一个内置属性 `[[Prototype]]`（在 ES5 中可通过 `__proto__ `访问，标准方法为`Object.getPrototypeOf()`），这个属性指向该对象的 “原型对象”（Prototype Object）。

例如，创建一个普通对象时，其原型默认指向 `Object.prototype`：

```js
const obj = {};
console.log(obj.__proto__ === Object.prototype); // true
console.log(Object.getPrototypeOf(obj) === Object.prototype); // true
```

什么是`Object.prototype`？

`Object.prototype` 是 JavaScript 里所有对象的"老祖宗"，原型链的顶层，再往上就 `null`了。

一张形象的图：

```text
太爷爷 (Object.prototype)
    ↑
    ├── 爷爷 (Array.prototype)
    ├── 爷爷 (Function.prototype)
    ├── 爷爷 (String.prototype)
    └── 爷爷 (其他各种原型)
           ↑
           ├── 爸爸 (某个构造函数.prototype)
           ├── 爸爸 (另一个构造函数.prototype)
                 ↑
                 我 (实例对象 p)
```

接下来分析样例代码：

`const obj = {};`这是js的创建一个对象的代码，可以向其中添加属性，当然如果直接在对象里面初始化也可以，看起来和python的字典很像。

`console.log(obj.__proto__ === Object.prototype);`这段代码是在控制台输出，`obj.__proto__ === Object.prototype`这是在进行判断，`obj.__proto__`这是获取对象的原型

#### 构造函数与prototype

构造函数（用于创建对象的函数）有一个特殊属性 prototype，它指向一个对象 ——该构造函数创建的所有实例的原型对象。（python的哲学是“一切皆对象”，虽然js不是这种哲学，但是在js中，很多东西都是对象，有的时候可以参照python的一些特性来理解js，因此构造函数本身也是一个对象，准确来说，在js中，**函数都是对象**，因此函数会有prototype属性，鼓噪函数的prototype属性就是由这个函数得来的所有实例的原型）

例如，`Array`是一个构造函数，其 `prototype` 是数组实例的原型：

```js
const arr = [1, 2, 3];
console.log(arr.__proto__ === Array.prototype); // true
console.log(Array.prototype.__proto__ === Object.prototype); // true
```

对于`console.log(Array.prototype.__proto__ === Object.prototype);`的解释：

前面已经说了，构造函数的`prototype`属性指向一个对象，而每一个对象都有一个内置属性`[[Prototype]]`，因此通过该对象的`__proto__`就能得到`Object.prototype`。

构造函数除了js内置的，还可以自己定义，用`function`关键字定义，定义方法和普通函数是一样的，但是在调用时有点区别。

```js
function Person(name, age){
    this.name = name;
  	this.age = age;
}
//构造函数调用
const p=new Person("小明",18);
//普通函数调用
Person("小明",18);
```

#### 原型链的形成

当访问一个对象的属性时，JavaScript 引擎会先在对象自身查找；若未找到，则沿着 `[[Prototype]]` 指向的原型对象查找；若仍未找到，继续沿着原型对象的` [[Prototype]] `向上查找，直到找到属性或抵达原型链的终点（`null`）。这一系列嵌套的原型关联，就构成了原型链。

原型链的终点是 `Object.prototype.__proto__`，其值为 `null`：

```js
console.log(Object.prototype.__proto__); // null
```

这里有一点会混淆，`Object.prototype.__proto__`能不能写成`Object.prototype.prototype`？

不能这么写，因为`prototype`是只有函数（普通函数和构造函数，除箭头函数外）才有的属性，但是所有对象都有`__proto__`属性。

关于node.js原型的一些基础知识先写到这，深入了解node.js原型链见文章[深入理解 Node.js 中的原型链在 JavaScript 及基于其构建的 Node.js 中，原型链（Prototy - 掘金](https://juejin.cn/post/7535805690712965156)

### node.js原型链污染

#### 漏洞原理

攻击者可通过修改对象的原型（`__proto__`或 `prototype`）来影响所有继承该原型的对象，从而改变程序逻辑甚至实现远程代码执行（RCE）。

核心原理： 当访问对象属性时，若该对象本身不存在该属性，JS 引擎会沿着原型链向上查找，直到 `Object.prototype` 或 `null`。如果攻击者能向原型链注入恶意属性，那么所有继承该原型的对象都会受影响。

基础样例：

```js
let a = {};
let b = {};
b.__proto__.isAdmin = true;
console.log(a.isAdmin); // true
```

讲解一下这段代码：

`let a = {};`这是声明了一个对象，`b.__proto__.isAdmin = true;`这里则是通过`b.__proto__`获取到b这个对象的原型，即`Object.prototype`,然后`.isAdmin`则是设置了一个`isAdmin`属性，并赋值为true。

这样由`Object`创建出来的所有对象都有这个属性了。

在 js 中每个函数都有一个 prototype 属性，而每个对象中也有一个 proto 属性用来指向实例对象的原型，而每个原型也都有一个 constructor 属性执行相关联的构造函数，我们就是通过构造函数生成实例化的对象。

![以图片的方式更好理解](https://wiki.wgpsec.org/images/js-prototype-chain-pollution/3.png)

（如果图片加载不出，点击https://wiki.wgpsec.org/images/js-prototype-chain-pollution/3.png）

关于原型链污染简单样例，见文章：

[深入理解 JavaScript Prototype 污染攻击 | 离别歌](https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html)

### node.js沙箱逃逸

#### 什么是沙箱

在Node.js中，沙箱是一种安全机制，用于在一个隔离的环境中运行不可信的代码，限制其访问宿主机器的敏感资源，如文件系统、网络、进程环境变量等。node.js有一个内置的vm模块来构建沙箱环境，node.js沙箱本质上是创建了一个context（上下文），上下文就是一个由V8引擎创建的、独立的全局执行环境，他和程序执行时的环境隔离开来，使在vm中的代码不会调用到外部的变量和函数，并且还限制了某些模块的导入和执行。

```js
const context = {};
const vm = require('vm');
vm.createContext(context);
const code = 'require(\'child_process\').exec("calc")';

vm.runInContext(code, context);
```

这段代码在沙箱内执行会报错。

先讲解一下这段代码的意思：

`const vm = require('vm');`这是导入vm模块并且保存在一个叫做vm的常量变量里面，因此vm这个变量就有vm模块的所有API。

`vm.createContext(context);`这是在使用vm模块的`createContext`方法，把context这个普通对象变成一个上下文隔离对象。

`const code = 'require(\'child_process\').exec("calc")';`这是导入了child_process模块并且调用exec方法执行系统命令，不过这只是一个字符串，并没有真的执行命令。

`vm.runInContext(code, context);`这是实际执行代码，`runInContext`这个方法用于在指定的上下文中运行代码，实际的意思就是：请在一个隔离的沙箱环境 context 中，执行 code 字符串里的 JavaScript 代码。

为什么会报错？

因为在这个沙箱中不存在require函数。

#### vm逃逸

##### 原型链逃逸

node.js沙箱报错后就会直接退出程序，这意味着在沙箱内执行了`process.exit()`结束了当前程序,也就是说`process`是外部和内部共有的一个模块，只要获取到process，就能获取到require模块，导入`child_process`。

```js
const vm = require("vm");

const context = {};

code =       //this获取当前沙箱的全局对象，也就是context
    `var exec = this.constructor.constructor;     
    var require = exec('return process.mainModule.constructor._load')();
    console.log(require('child_process').execSync("calc").toString());`

vm.runInNewContext(code,context);
```

`this.constructor.constructor`这段代码是在获取`Object`构造函数，根据上面的图片会有一个疑惑，为什么图中是通过原型对象获取的构造函数，而这里是通过实例函数直接获取到的？

联想一下js的属性查找机制就知道了，当对象自身没有该属性时会向原型查找，而原型有`constructor`属性，因此`obj.constructor`和`obj.__proto__.constructor`本质上是一样的，只是省略了一个步骤而已。

这里给一个简单的沙箱逃逸的payload：

```js
const result = this.constructor.constructor('return process')()
    .mainModule.require('child_process')
    .execSync('cat /flag') 
    .toString();
console.log(result);
```

`this.constructor`这一步获取到构造函数`Object`，`this.constructor.constructor`这一步则是获取`Object`这个构造函数的构造函数，**所有函数都有一个构造函数，即`Function`**，这个构造函数有个特点，由`Function`构造函数创建的函数，其作用域是全局作用域，因此`this.constructor.constructor('return process')`得到的是沙箱外部的process对象。而这条语句实际上创建了一个匿名函数（这由Function语法决定）,后面加上了一个括号是让这个匿名函数立即执行。

`process.mainModule.require`这条语句则是获取了当前应用主模块的reqiure函数，能够加载所有模块。

`require('child_process').execSync('cat /flag')`则是导入child_process模块并使用其中的execSync函数执行系统命令，从而控制服务器。

`.toString();`则是把命令执行的结果变成字符串，便于输出。

关于其他沙箱逃逸的知识，由于我暂时没碰到，所以先放一篇文章在这，等遇到了再学。见[node.js 原型链污染与沙箱逃逸总结 - Litsasuk - 博客园](https://www.cnblogs.com/Litsasuk/articles/18823416#1漏洞原理-1)
