---
title: SSH穿越跳板机登录远程服务器
date: 2016-06-30 17:59:56
tags: [linux, ssh]
---

公司出于安全考虑，登录业务服务器之前必须先登录到跳板机然后再通过跳板机登录业务服务器，本来流程也不算太复杂，但是作为一线攻城狮登录业务服务器的频率高，显然不能忍这种操作方式。于是在保证安全性的同时，必须想办法提高工作效率了。

## 蛋疼的登录方式

1.登录跳板机

```bash
# ~/.zshrc
alias gojump="ssh -p 1234 -i ~/.ssh/server_rsa wangpeng@jump.company.com"
$ gojump
```

2.登录业务服务器

```bash
$ ssh -p 1234 wangpeng@123.123.123.123
```

## 使用ProxyCommand的登录方式

编辑`~/.ssh/config`，增加如下配置。

```bash
Host 123.123.123.123
	IdentityFile ~/.ssh/server_rsa
	User wangpeng
	Port 1234
	ForwardAgent yes
	ProxyCommand ssh -p 1234 wangpeng@jump.company.com -W %h:%p 2> /dev/null
```

<!--more-->

保存后，就可以直接通过`ssh 123.123.123.123`的方式登录远程业务服务器了，简直方便！

## 优化ProxyCommand登录方式

当你有多个业务服务器需要管理的时候（没办法，你是一线攻城狮），挨个给远程机器配置ProxyCommand也是一件蛋疼的事，而且增加机器的时候还得增加配置！其实不必这么麻烦，`Host`项是支持正则表达式配置的（好像openssh5.5以上版本支持），于是优化后的ProxyCommand配置就像下面这样。

```bash
Host test*
	IdentityFile ~/.ssh/server_rsa
	User wangpeng
	Port 1234
	ForwardAgent yes
	ProxyCommand ssh -p 1234 wangpeng@jump.company.com -W $(echo %h|awk -F 'test' '{print $2}'):%p 2> /dev/null
```

然后就可以通过这样登录：

```bash
$ ssh test123.123.123.123
```

增加机器也不需要修改ssh配置了:D
