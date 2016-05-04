---
title: wordpress xmlrpc攻击导致502 Bad Gateway
date: 2016-04-30 21:39:26
tags: [安全]
---

都说玩博客的都要经历三个阶段：
> 1.QQ空间、CSDN、iteye等等博客网站上写；
> 
> 2.觉得都在第三方网站上写不爽，得自己折腾，于是自购服务器空间域名，搭建wordpress博客；
>
> 3.慢慢发现wordpress上写博客各种不爽，发现了jekyll、hexo之类高大上的开源博客系统，结合github pages+markdown，原来这才是真爱！

这里记录的是之前折腾wordpress时遇到的xmlrpc攻击问题。

最近刚刚搞好博客的备案，准备开始倒腾自己的网站了，结果今天一上来发现502 Bad Gateway！

<!--more-->

心想ECS不会这么不靠谱吧，赶紧上ECS后台查了下服务器状况，发现除了2起远程代码攻击被云盾拦截了之外没什么特别的异常，云盾还是挺给力的；

看来只能登录到ECS服务器上查个究竟了，检查服务器负载和内存都挺正常，nginx和php的进程也没什么异样；
重启大法！重启nginx和php进程，发现博客能访问了，于是也没多想接着倒腾，可是过了不到一分钟，访问速度变得异常缓慢，接着又502了…

怀疑是php和nginx的配置有问题，于是增加了一些优化的配置项，重启，依旧是1分钟后502…

继续检查php，nginx配置，发现/etc/php5/fpm/pool.d/www.conf中

```
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
```

php5-fpm默认启动2个进程，最多启动5个进程，但是之前重启php之后直接就到5个进程了，应该是有异常的访问；

检查nginx的access log发现果然刷刷的很多不明ip访问记录:
46.166.139.20在不断的发起xmlrpc的post请求，google后才知道原来是wordpress xmlrpc攻击！

原来wordpress在3.5版本之后默认开启了xmlrpc功能，而且没有提供关闭该功能的设置！XML-RPC 是 WordPress 用于第三方客户端（如 WordPress iPhone 和安卓客户端，Windows Live Writer 等）的 API 接口，还可以用于 pingbacks 和 trackbacks 端口，作为站点之间的通讯桥梁。但是xmlrpc功能存在可能被黑客攻击破解的漏洞，如果用不到第三方客户端的话，建议关闭掉这个选项。

由于后台没有提供关闭xmlrpc功能的选项，所以只能通过修改当前主题的functions.php文件来禁用xmlrpc功能。在functions.php中添加：

```
/*禁用xmlrpc,防止攻击*/
add_filter('xmlrpc_enabled', '__return_false');
```

添加之后发现恶意ip的xmlrpc访问已经返回499状态码了。

或者直接禁掉恶意ip的访问。

```
iptables -I INPUT -s 46.166.139.20 -j DROP
```

access log一下子清静了。又能愉快的折腾了。
