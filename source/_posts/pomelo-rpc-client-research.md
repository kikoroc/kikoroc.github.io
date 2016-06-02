---
title: pomelo-rpc原理解析之client
date: 2016-05-31 15:44:51
tags: [nodejs, pomelo]
---

[pomelo-rpc](https://github.com/NetEase/pomelo-rpc/)是pomelo项目底层的rpc框架，提供了一个多服务器进程间进行rpc调用的基础设施。 pomelo-rpc分为客户端和服务器端两个部分。 客户端部分提供了rpc代理生成，消息路由和网络通讯等功能，并支持动态添加代理和远程服务器配置。 服务器端提供了远程服务暴露，请求派发，网络通讯等功能。

本文主要分析pomelo-rpc中client部分的实现原理以及运行逻辑。

## Client初始化

### Client.create(opts)

返回一个rpc client实例，并创建一个client station，在station中会创建基于tcp/ws的mailbox用于底层的网络通讯。  
初始化过程很简单，具体可以查阅源码。  
这里有两个概念需要理解一下：  
>**MailBox**:mailbox其实就是rpc client到远程服务的一个socket连接，提供心跳和数据转发的底层网络通讯功能；  
**MailStation**:MailStation负责维护当前rpc client与多个远程服务的socket连接，也就是说mailbox都在MailStation里面，而且消息路由也是由MailStation来实现。

<!--more-->

## 连接remote server

pomelo proxy组件在完成rpc client的初始化之后，会监听集群服务器的上下线和更新的事件，然后分别处理对应的事件。

```javascript
  // rpc client初始化
  this.client = genRpcClient(this.app, opts);
  // 监听servers事件
  this.app.event.on(events.ADD_SERVERS, this.addServers.bind(this));
  this.app.event.on(events.REMOVE_SERVERS, this.removeServers.bind(this));
  this.app.event.on(events.REPLACE_SERVERS, this.replaceServers.bind(this));
```  

### addServers(servers)

proxy.addServers()

```javascript
pro.addServers = function(servers) {
  if (!servers || !servers.length) {
    return;
  }
  // 生成服务器的代理，生成代理的原理跟rpc-server差不多
  // 具体参考http://kikoroc.com/2016/05/25/pomelo-rpc-server-research.html
  // 这里就不重复介绍了
  // 不过这里还会调用client.addProxy()来生成代理
  genProxies(this.client, this.app, servers);
  this.client.addServers(servers);
};
```

生成代理之后，会给app设置rpc和sysrpc属性，以及直接的rpcInvoke调用。

```javascript
  this.app.__defineGetter__('rpc', function() {
    return self.client.proxies.user;
  });
  this.app.__defineGetter__('sysrpc', function() {
    return self.client.proxies.sys;
  });
  this.app.set('rpcInvoke', this.client.rpcInvoke.bind(this.client), true);
```

所以在pomelo中我们发起rpc调用都是如下形式：

```javascript
  this.app.rpc.yourServerType.yourServerTypeRemote.method()
```

client.addServers()  
client.addServers()实际调用station.addServers()  

```javascript
pro.addServer = function(serverInfo) {
  if(!serverInfo || !serverInfo.id) {
    return;
  }

  var id = serverInfo.id;
  var type = serverInfo.serverType;
  this.servers[id] = serverInfo;
  this.onlines[id] = 1;

  if(!this.serversMap[type]) {
    this.serversMap[type] = [];
  }
  // 加入到serversMap中
  this.serversMap[type].push(id);
  this.emit('addServer', id);
};
```

然后station建立跟remote server的socket链接（mailbox）。

## rpc invoke

在pomelo中我们发起rpc调用的方式主要包括：

```javascript
this.app.rpc.yourServerType.yourServerTypeRemote.method(args, cb)
```
or
```javascript
this.app.rpcInvoke(sid, msg, cb)
```

实际处理逻辑就是调用rpc client的proxy，而proxy呢则实际就是调用station.dispatch()来分发消息，station根据serverId和dispatch策略来调用对应的mailbox（socket链接）来转发数据。这样就完成了rpc调用的过程。

## 其他

### lazyConnect

pomelo proxy可以配置lazyConnect属性，意思是addServer的时候并没有建立跟remote server的连接，而是当有请求到对应remote server的时候再建立跟remote server的连接。个人觉得似乎没有必要~

### pendingSize

在MailStation中，有pendings和pendingSize属性，意思是当请求到某remote server的mailbox还在connecting或者还没有connected的时候，MailStation会帮助缓存rpc请求到pindings中，当mailbox connected之后再一起flush到mailbox中，这样可以防止server重启时丢掉rpc请求，而pendingSize则可以指定缓存请求的条数。

个人觉得当remote server进程数较少的时候，这种缓存的机制还是挺不错的，当进程数较多的时候，除非重启全部serverType的进程，不然rpc请求会转发给其他connected的rpc server处理。当然这种机制的存在还是很不错的。
