---
title: Ubuntu安装docker
date: 2016-07-21 21:13:30
tags: [docker, linux]
---

### 支持的Ubuntu版本

Docker目前支持以下Ubuntu版本：
- Ubuntu Xenial 16.04 (LTS)
- Ubuntu Wily 15.10
- Ubuntu Trusty 14.04 (LTS)
- Ubuntu Precise 12.04 (LTS)

### 系统条件

安装Docker需要以下系统条件：
- 64位系统
- 系统内核需要3.10以上，如果是Ubuntu 12.04需要3.13以上

可以通过以下命令确认系统版本以及内核版本：
```bash
$ uname -r
3.13.0-32-generic
$ cat /etc/issue
Ubuntu 14.04.2 LTS \n \l
```

<!--more-->

### 更新APT源

1. Docker安装过程中需要`apt-transport-https`库，安装
```bash
$ sudo apt-get update
$ sudo apt-get install apt-transport-https ca-certificates
```

2. 添加新的GPG key
```bash
$ sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
```

3. 修改docker.list
打开`/etc/apt/sources.list.d/docker.list`文件（如果没有就新建这个文件），清除文件内容，然后添加以下内容：

> Ubuntu Precise 12.04 (LTS)
```bash
deb https://apt.dockerproject.org/repo ubuntu-precise main
```
> Ubuntu Trusty 14.04 (LTS)
```bash
deb https://apt.dockerproject.org/repo ubuntu-trusty main
```
> Ubuntu Wily 15.10
```bash
deb https://apt.dockerproject.org/repo ubuntu-wily main
```
> Ubuntu Xenial 16.04 (LTS)
```bash
deb https://apt.dockerproject.org/repo ubuntu-xenial main
```
保存修改。

4. 更新APT索引
```bash
$ sudo apt-get update
```

5. 如果之前安装过docker，先卸载
```bash
$ sudo apt-get purge lxc-docker
```

6. 检查APT设置是否正确
```bash
$ apt-cache policy docker-engine
docker-engine:
  Installed: 1.11.2-0~trusty
  Candidate: 1.11.2-0~trusty
  Version table:
 *** 1.11.2-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
        100 /var/lib/dpkg/status
     1.11.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.11.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.10.3-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.10.2-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.10.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.10.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.9.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.9.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.8.3-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.8.2-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.8.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.8.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.7.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.7.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.6.2-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.6.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.6.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
     1.5.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
```

7. 安装其他依赖
```bash
$ sudo apt-get update
$ sudo apt-get install linux-image-extra-$(uname -r)
```

如果是Ubuntu 12.04或者14.04，还需要安装`apparmor`，`apt-get install apparmor`。

### 安装启动Docker服务

1. 安装docker
```bash
$ sudo apt-get update
$ sudo apt-get install docker-engine
```

2. 启动docker daemon进程
```bash
$ sudo service docker start
```

至此完成docker在ubuntu系统上的安装。

### 其他

解决 *Cannot connect to the Docker daemon. Is the docker daemon running on this host?* 的报错：
`sudo usermod -aG docker $(whoami)`
