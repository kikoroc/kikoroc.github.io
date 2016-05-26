---
title: pomelo-rpc原理解析之server
date: 2016-05-25 17:59:14
tags: [nodejs, pomelo]
---

[pomelo-rpc](https://github.com/NetEase/pomelo-rpc/)是pomelo项目底层的rpc框架，提供了一个多服务器进程间进行rpc调用的基础设施。 pomelo-rpc分为客户端和服务器端两个部分。 客户端部分提供了rpc代理生成，消息路由和网络通讯等功能，并支持动态添加代理和远程服务器配置。 服务器端提供了远程服务暴露，请求派发，网络通讯等功能。

本文主要分析pomelo-rpc中server部分的实现原理以及运行逻辑。

## server初始化

### Server.create(opts)

创建一个rpc server实例。根据配置信息加载远程服务代码，并生成底层acceptor。  
首先看create部分源码：  

```javascript
module.exports.create = function(opts) {
  if(!opts || !opts.port || opts.port < 0 || !opts.paths) {
    throw new Error('opts.port or opts.paths invalid.');
  }
  // 根据paths加载远程服务
  var services = loadRemoteServices(opts.paths, opts.context);
  opts.services = services;
  var gateway = Gateway.create(opts);
  return gateway;
};
```

首先`loadRemoteServices()`方法根据opts参数中的paths加载远程服务。
其中pomelo paths的格式类似(pomelo根据约定封装,详见https://github.com/NetEase/pomelo/blob/master/lib/components/remote.js)：

```json
[
  {
    "namespace": "user",
    "serverType": "test",
    "path": "/data/pomelo/app/servers/test/remote/"
  },
  {
    "namespace": "sys",
    "serverType": "test",
    "path": "/data/pomelo/pomelo/lib/common/remote/backend/"
  }
]
```

<!--more-->

```javascript
var loadRemoteServices = function(paths, context) {
  var res = {}, item, m;
  for(var i=0, l=paths.length; i<l; i++) {
    item = paths[i];
    // Loader是pomelo-loader,用来加载pomelo handler和remote服务
    // 关于Loader的细节可以参考https://github.com/NetEase/pomelo-loader/
    m = Loader.load(item.path, context);

    if(m) {
      createNamespace(item.namespace, res);
      for(var s in m) {
        res[item.namespace][s] = m[s];
      }
    }
  }

  return res;
};

var createNamespace = function(namespace, proxies) {
  proxies[namespace] = proxies[namespace] || {};
};
```

`loadRemoteServices()`最终得到的services对象类似如下结构的数据：

```json
{
  "user": {
    "testRemote": require("/path/to/testRemote.js"),
    "test2Remote": require("/path/to/test2Remote.js"),
    ...
  },
  "sys": {
    "msgRemote": require("/data/pomelo/pomelo/lib/common/remote/backend/msgRemote.js")
  }
}
```

### Gateway.create(opts)

创建gateway对象，并初始化dispatcher和acceptor。  
gateway的构造方法：

```javascript
var Gateway = function(opts) {
  EventEmitter.call(this);
  this.opts = opts || {};
  this.port = opts.port || 3050;
  this.started = false;
  this.stoped = false;
  this.services = opts.services;
  if(!!this.opts.reloadRemotes) {
    // 如果remote配置reloadRemotes=true的话，gateway会通过`fs.watch()`来检测remote文件的变化，
    // 然后通过pomelo-loader重新require对应的remote文件来实现remote的热更新。
    watchServices(this, dispatcher);
  }
  var self = this;

  this.acceptors = {};
  // __defineGetter__的用法可以参考
  // https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/__defineGetter__
  this.acceptors.__defineGetter__('tcp', utils.load.bind(null, '../rpc-server/acceptors/tcp-acceptor'));
  this.acceptors.__defineGetter__('ws', utils.load.bind(null,'../rpc-server/acceptors/ws-acceptor'));

  if(!!opts.acceptorName && opts.acceptorName === 'ws') {
    this.acceptorFactory = this.acceptors.ws;
  } else {
    // 默认是使用tcp协议的acceptors
    this.acceptorFactory = this.acceptors.tcp;
  }

  if(!!opts.acceptorFactory) {
    this.acceptorFactory = opts.acceptorFactory;
  }
  // Dispatcher初始化，没什么好看的
  var dispatcher = new Dispatcher(this.services);
  // acceptorFactory.create初始化acceptor，也没什么好看的
  this.acceptor = this.acceptorFactory.create(opts, function(tracer, msg, cb) {
    dispatcher.route(tracer, msg, cb);
  });
};
```

### Dispatcher.route(tracer, msg, cb)

提供路由服务，路由消息到对应的service。

```javascript
/**
 * route the msg to appropriate service object
 *
 * @param msg msg package {service:serviceString, method:methodString, args:[]}
 * @param services services object collection, such as {service1: serviceObj1, service2: serviceObj2}
 * @param cb(...) callback function that should be invoked as soon as the rpc finished
 */
pro.route = function(tracer, msg, cb) {
  tracer.info('server', __filename, 'route', 'route messsage to appropriate service object');
  // this.services 就是loadRemoteServices()加载的本地remote service
  var namespace = this.services[msg.namespace];
  if(!namespace) {
    tracer.error('server', __filename, 'route', 'no such namespace:' + msg.namespace);
    utils.invokeCallback(cb, new Error('no such namespace:' + msg.namespace));
    return;
  }

  var service = namespace[msg.service];
  if(!service) {
    tracer.error('server', __filename, 'route', 'no such service:' + msg.service);
    utils.invokeCallback(cb, new Error('no such service:' + msg.service));
    return;
  }

  var method = service[msg.method];
  if(!method) {
    tracer.error('server', __filename, 'route', 'no such method:' + msg.method);
    utils.invokeCallback(cb, new Error('no such method:' + msg.method));
    return;
  }

  var args = msg.args.slice(0);
  args.push(cb);
  // 调用remote service方法
  method.apply(service, args);
};
```

*至此，pomelo-rpc的server初始化工作就完成了，接下来就是启动server。*

## server启动

### Gateway.start()

```javascript
pro.start = function() {
  if(this.started) {
    throw new Error('gateway already start.');
  }
  this.started = true;

  var self = this;
  this.acceptor.on('error', self.emit.bind(self, 'error'));
  this.acceptor.on('closed', self.emit.bind(self, 'closed'));
  // 启动acceptor并监听port端口
  this.acceptor.listen(this.port);
};
```

### Acceptor.listen(port)

启动acceptor并监听port端口，这里以tcp acceptor为例。

```javascript
pro.listen = function(port) {
  //check status
  if(!!this.inited) {
    utils.invokeCallback(this.cb, new Error('already inited.'));
    return;
  }
  this.inited = true;

  var self = this;

  this.server = net.createServer();
  this.server.listen(port);

  this.server.on('error', function(err) {
    logger.error('rpc server is error: %j', err.stack);
    self.emit('error', err, this);
  });
  // 处理链接请求
  this.server.on('connection', function(socket) {
    // 设置socket自增id
    socket.id = self.socketId++;
    // 保存socket句柄到本地sockets
    self.sockets[socket.id] = socket;
    // 设置socket的消息编解码处理器，处理各种类型的消息（ping,pong,msg等）
    // pkgSize默认-1，不限制消息长度
    socket.composer = new Composer({maxLength: self.pkgSize});
    // 心跳超时检测timer
    self.timer[socket.id] = null;
    // 启动心跳检测，如果heartbeat timeout间隔内没有心跳包过来，就断开连接。
    // 下面socket.composer.on('data' ...中的self.heartbeat才是真正连接后的心跳
    // 相当于这里只是处理初始连接时的心跳
    self.heartbeat(socket.id);

    socket.on('data', function(data) {
      // 调用composer解析数据流
      // feed读完数据后会emit data事件
      socket.composer.feed(data);
    });
    // 接收feed emit的data事件
    socket.composer.on('data', function(data) {
      self.heartbeat(socket.id);
      if(data[0] === PING) {
        //incoming::ping，response with PONG
        socket.write(socket.composer.compose(PONG));
      } else {
        try {
          var pkg = JSON.parse(data.toString('utf-8', 1));
          var id  = null;
          // 处理消息
          if(pkg instanceof Array) {
            processMsgs(socket, self, pkg, id);
          } else {
            processMsg(socket, self, pkg, id);
          }
        } catch(err) { //json parse exception
          if(err) {
            // 重置编解器状态
            socket.composer.reset();
            logger.error(err);
          }
        }
      }
    });

    socket.on('error', function(err) {
      logger.error('[pomelo-rpc] tcp socket error: %j', err);
    });

    socket.on('close', function() {
      logger.error('[pomelo-rpc] tcp socket close: %s', socket.id);
      delete self.sockets[socket.id];
      delete self.msgQueues[socket.id];
      if(self.timer[socket.id]){
        clearTimeout(self.timer[socket.id]);
      }
      delete self.timer[socket.id];
    });
  });
  // 定时flush缓存的消息数据
  if(this.bufferMsg) {
    this._interval = setInterval(function() {
      flush(self);
    }, this.interval);
  }
};
```

### processMsg()

处理rpc消息。实际就是调用dispatcher.route()来根据namespace和service来调用对应的remote method。

```javascript
var processMsg = function(socket, acceptor, pkg, id) {
  var tracer = new Tracer(acceptor.rpcLogger, acceptor.rpcDebugLog, pkg.remote, pkg.source, pkg.msg, pkg.traceId, pkg.seqId);
  tracer.info('server', __filename, 'processMsg', 'tcp-acceptor receive message and try to process message');
  // 实际就是调用dispatcher.route()
  acceptor.cb.call(null, tracer, pkg.msg, function() {
    var args = Array.prototype.slice.call(arguments, 0);
    for(var i=0, l=args.length; i<l; i++) {
      if(args[i] instanceof Error) {
        args[i] = cloneError(args[i]);
      }
    }
    var resp;
    if(tracer.isEnabled) {
      resp = {traceId: tracer.id, seqId: tracer.seq, source: tracer.source, id: pkg.id, resp: Array.prototype.slice.call(args, 0)};
    }
    else {
      resp = {id: pkg.id, resp: Array.prototype.slice.call(args, 0)};
    }
    if(acceptor.bufferMsg) {
      // 如果开启缓冲消息则将结果缓存到队列，由_interval定时flush，用以减少网络io次数
      enqueue(socket, acceptor, resp);
    } else {
      // 发送封装remote方法的返回结果
      socket.write(socket.composer.compose(RES_TYPE, JSON.stringify(resp), id));
    }
  });
};
```

## 总结

通过对pomelo-rpc server部分代码的分析，可以很清晰了解到server端主要作用就是暴露远程服务（remote目录下的.js文件）、根据消息的namespace和service信息派发到对应的remote服务处理、基于tcp/ws来提供底层的网络通讯。

在日常开发中，新手很容易遇到rpc调用超时的情况，一般来看都是因为某些remote方法没有正确回调或者根本漏写了回调。
