---
title: 解决hexo神烦的DTraceProviderBindings MODULE_NOT_FOUND
date: 2016-05-04 22:28:37
tags: [hexo, nodejs]
---

今晚折腾hexo的时候遇到很恶心的hexo报错，执行任何hexo命令都会报如下错误。
## hexo报错
```
{ [Error: Cannot find module './build/Release/DTraceProviderBindings'] code: 'MODULE_NOT_FOUND' }
{ [Error: Cannot find module './build/default/DTraceProviderBindings'] code: 'MODULE_NOT_FOUND' }
{ [Error: Cannot find module './build/Debug/DTraceProviderBindings'] code: 'MODULE_NOT_FOUND' }
```

## 解决办法on google
google之后发现被这个问题困扰的小伙伴还真不少，网上最靠谱的解决办法是：
```
$ npm install hexo --no-optional
```
<!--more-->
的确很多小伙伴执行上面的命令之后都解决了报错的问题，但也有很多小伙伴跟我一样执行后没有任何效果，我甚至尝试了卸载后重装的方式：
```
$ npm uninstall hexo
$ npm install hexo --no-optional
```
可是依旧没有效果。
其实hexo暂时并没有用到`dtrace-prodider`，仅仅是报错而已，hexo的命令还是能执行，但对于强迫症来说，简直无法忍受。

## 我的解决办法
一番折腾之后，我找到了适合我的解决办法，重装`hexo-cli`:
```
$ npm uninstall hexo-cli -g
$ npm install hexo-cli -g
```
再来执行hexo命令果然没有报错信息了:D
