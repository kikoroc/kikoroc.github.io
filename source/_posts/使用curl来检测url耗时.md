---
title: 使用curl来检测url耗时
date: 2016-05-04 11:20:54
tags: [linux]
---

有时候遇到网站或者api访问速度慢，排除了服务器以及程序性能问题后，很可能是网络导致的慢速。我们可以通过linux中的curl命令来测试url的各项耗时。
```
curl -o /dev/null -s -w %{time_connect}:%{time_starttransfer}:%{time_total}:%{time_namelookup}:%{speed_download} http://url.com/api
```

>**time_connect**:建立到服务器的TCP连接所用的时间  
**time_starttransfer**:在发出请求后，web服务器返回数据的第一个字节所用的时间  
**time_total**:完成请求所用的时间  
**time_namelookup**:DNS解析时间，从请求开始到DNS解析完毕所用时间（记得关掉Linux的nscd的服务测试）  
**speed_download**:下载速度，单位字节每秒

另外可以配合
```
time wget http://url.com/api
```  
进一步测试。
