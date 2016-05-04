---
title: 为QC.Login.showPopup增加回调函数
date: 2014-09-22 15:38:45
tags: [qqsdk, javascript]
---

### 问题

今天开始接入qq的帐号登录功能了，主要是web端接入，使用腾讯开发平台提供的[js-sdk](http://wiki.connect.qq.com/js_sdk%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E)，在文档在发现一个很好用的api，`QC.Login.showPopup()`这个接口能直接打开qq登录的弹窗，用法也很简单：

```
QC.Login.showPopup({
   appId:"11111111",
   redirectURI:"http://iampeng.wang/"
});
```

实际使用中，发现需要一个登录成功后的回调函数，但是`QC.Login.showPopuo()`接口并没有提供任何回调函数。

### 思路

在剩下的文档中，发现`QC.Login()`这个接口是提供回调函数的，用法如下：

```
QC.Login({
    btnId:"loginBtn"
},loginCallback,logoutCallback);
```
<!--more-->

`QC.Login()`接口分别提供了两个可选的回调函数，`loginCallback`是登录成功后的回调，`logoutCallback`是注销登录成功后的回调。
那么，可不可以把`QC.Login()`和`QC.Login.showPopup()`这两个接口组合起来实现呢？

> 在腾讯开发平台的[js-sdk](http://wiki.connect.qq.com/js_sdk%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E)中提到：**如果已使用QC.Login方法，则不需再使用此方法（QC.Login.showPopup）**。

虽然是这么说，但是貌似还只有这么做才行，接下来的做法就是：

```
QC.Login({},function(data,ops){
	// your callback here.
	console.log("login success!");
});
QC.Login.showPopup({
	appId:"111111"
});
```

运行后发现完美解决问题：）
虽然暂时还没来得及看js-sdk的源码，但基本猜测得到原理，`QC.Login.showPopup`只是提供直接提供一个登录的弹窗，具体的登录逻辑肯定还是调用`QC.Login`来实现的，所以在`QC.Login`中添加登录成功的回调函数同样有效。等有时间再一探究竟~
