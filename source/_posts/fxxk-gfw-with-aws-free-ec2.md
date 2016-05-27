---
title: Amazon AWS免费套餐搭建vpn/ss服务
date: 2016-05-26 15:36:07
tags: [linux]
---

翻墙作为现在程序猿们的必修课，大家十八般武艺各种翻，我也尝试了很多方法，从最开始朋友给的vpn账号，到购买简单易用的插件服务，曲径红杏什么的，接着购买各种付费ss，vpn，到现在折腾亚马逊一年免费的ec2套餐自己搭建vpn/ss服务，一来基本免费，二来可以自己动手搭建vpn/ss服务折腾一下也挺爽。

![aws](http://kikoroc.qiniudn.com/aws.png)

开通amazon aws/ec2的教程网上很多，我这里就不详细写了，大家自己google，有几点需要注意的简单提一下：

> 1.去https://aws.amazon.com/ 注册账号，你amazon.cn的账号并不能用；  
2.需要有一张可用的信用卡，开通的过程中会需要扣你1美刀的预付款作为验证信用卡可用性，验证过程是电话验证，所以还要保持你的手机可用；  
3.进入控制台之后，可能会纠结在哪个区申请ec2实例，可以去http://www.cloudping.info/ 测试你到哪个机房延迟比较低，一般来说东京延迟最低；  
4.创建ec2的过程基本都是一路下一步，注意看清楚'free'标识就行；  
5.保存好生成的密钥（PEM）文件，以后ssh要用到。  

ok，至此你的一年免费ec2实例就申请好了。至于一年后，该付费付费，还想继续吃免费午餐的话重新注册一个新账号吧......
以下都以Ubuntu Server为例。

<!--more-->

### 登录到ec2实例

```bash
ssh -i "/path/to/your/pem/file" ubuntu@your.ec2.instance.at.compute.amazonaws.com
```

### 搭建vpn服务

vpn具有先天跨平台的优势，各类操作系统都有自己的vpn功能，直接可以用，无需第三方客户端。  

但是vpn不够灵活，默认情况下，系统所有的流量都会从VPN走，很多时候都是不必要的，比如你上个youku，数据是从youku到东京机房，再从东京机房回到你的电脑，绕一大圈，不仅速度慢还浪费VPN流量。当然你也可以手动改路由表，自己决定哪些流量走VPN哪些直连，但是这个工作量太大了。

不过VPN够全能，ssh/ss做不了的VPN全都没问题，比如你安装了twitter的客户端，ssh/ss就没办法帮你翻了，但是VPN没问题。
所以，VPN服务还是选择安装了。VPN包含PPTP,L2TP,OpenVPN,IPSec四种模式，这里我们选择安全性和实用性都ok的PPTP模式。

* 安装pptp  

```bash
sudo apt-get install pptpd
```

* 修改`pptpd.conf`默认配置

```bash
sudo vim /etc/pptpd.conf
#在最后添加
localip 192.168.0.1
remoteip 192.168.0.234-238,192.168.0.245
```

* 添加vpn账号

```bash
sudo vim /etc/ppp/chap-secrets
#添加
# Secrets for authentication using CHAP
# client        server  secret                  IP addresses
kikoroc         pptpd   yourpassword            *
```

* 设置vpn dns

```bash
sudo vim /etc/ppp/pptpd-options
#添加
ms-dns 8.8.8.8
ms-dns 8.8.4.4
#使用google dns
```

* 打开ipv4 forward

```bash
sudo vim /etc/sysctl.conf
#打开注释
net.ipv4.ip_forward=1

sudo sysctl -p
```

* 添加iptables规则

```bash
sudo apt-get install iptables
sudo iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eth0 -j MASQUERADE

/etc/init.d/iptables save
iptables restart
```

* 重启pptpd服务

```bash
sudo service pptpd restart
```

or

```bash
sudo /etc/init.d/pptpd restart
```

至此，VPN服务就搭建完成了。可以使用系统自带的VPN工具试试链接了。

* 查看当前有哪些VPN连接到服务器

```bash
last | grep still | grep ppp
```

### 搭建shadowsocks服务

前面说到vpn够全能但不够灵活，那shadowsocks可谓瑞士军刀般小巧灵活了。因为shadowsocks属于HTTP/SOCK5代理，配合浏览器插件（SwitchySharp等）很方便。shadowsocks相关的资料可以参考https://github.com/shadowsocks/shadowsocks/wiki/

* 安装shadowsocks

```bash
sudo apt-get install python-pip
sudo pip install shadowsocks
```

* 配置shadowsocks文件

```bash
sudo vim /etc/shadowsocks.json

{
    "server":"yourinstance.compute.amazonaws.com",
    "server_port":8388,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"yourpassword",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": true
}
```

* 打开tcp_fastopen

server和client如果都是linux3.7+，打开tcp_fastopen可以减小延时。

```bash
sudo echo 3 > /proc/sys/net/ipv4/tcp_fastopen
```

* 其他优化

可以参考https://github.com/shadowsocks/shadowsocks/wiki/Optimizing-Shadowsocks

* 启动shadowsocks

```bash
ssserver -c /etc/shadowsocks.json -d start
#查看shadowsocks日志
sudo tail -f /var/log/shadowsocks.log
```

至此，shadowsocks服务就搭建ok了，去https://github.com/shadowsocks/shadowsocks/wiki/Ports-and-Clients 下载对应的shadowsocks客户端开始体验墙外的酸爽吧~手机上同样可以体验~~

### 其他

网上很多教程漏了一点，要登录到ec2控制台开启vpn和shadowsocks的端口权限。

![端口设置](http://kikoroc.qiniudn.com/awsportsetting.png)
