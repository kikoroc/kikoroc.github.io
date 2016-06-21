---
title: qpsFilter来统计pomelo qps数据
date: 2016-06-21 16:01:18
tags: [nodejs, pomelo]
---

在pomelo中可以通过编写自己的filter来实现一些敏感词过滤，qps统计，统一参数检查等逻辑。本文是一个用于统计qps的filter，可以统计整个pomelo集群各个api的qps情况。

## qpsFilter.js

```javascript
/**
 * Created by kikoroc on 16/6/6.
 */
var logger = require('pomelo-logger').getLogger(__filename);
var moment = require('moment');
var redis = require('./path/to/your/redisInstance');

module.exports = function(opts){
    return new Filter(opts);
};

var qpsKey = function(){
    return 'QPS@'+moment().format('YYYYMMDDHH');
};

var Filter = function(opts){
    this.routes = {};
    this.stat = (function(self){
        setInterval(function(){
            var cmds = [];
            var key = qpsKey();
            for(var r in self.routes){
                cmds.push(['hincrby', key, r, self.routes[r]]);
            }
            logger.info('[qpsFilter]->', JSON.stringify(cmds));
            if(cmds.length>0){
                redis.multi(cmds).exec(function(err, replies) {
                    if(!!err){
                        logger.error('error@qpsFilter', err);
                    }
                    // 一周后过期
                    redis.expire(key, 7*24*60*60, function(err, ret){
                        if(!!err){
                            logger.error('error@qpsFilter', err);
                        }
                    });
                });
            }
            // reset
            self.routes = {};
        }, opts.interval*1000)
    })(this);
};

Filter.prototype.before = function(msg, session, next){
    if(msg.__route__){
        if(this.routes.hasOwnProperty(msg.__route__)){
            this.routes[msg.__route__] += 1;
        }else{
            this.routes[msg.__route__] = 1;
        }
    }
    next();
};
```
<!--more-->

可以按小时的粒度来统计各个api的请求次数，并保存到redis中。

## usage

```javascript
// app.js
var qpsFilter = require('./path/to/filter/qpsFilter');

app.configure('production|development', function(){
  app.before(qpsFilter({
      interval: 60 // 60s的持久化间隔
  }));
});  
```

## query

可以提供http api来查询qps数据。

```javascript
// http api
var date = req.query.qpsDate.trim();
// 默认当前小时
if(date==''){
    date = moment().format('YYYYMMDDHH');
}else {
    if(!/^[0-9]{6}$/.test(date)){
        res.send({code: 500, msg: 'date format error.'});
    }else{
        // 自动补年份
        date = moment().format('YYYY')+date;
    }
}
var sec = 60*60;
if(date==moment().format('YYYYMMDDHH')){
    // 当前小时的话，重新计算秒数
    sec = Number(moment().format('mm'))*60;
}
qpsRedis.hgetall('QPS@'+date, function(err, ret){
    if(!!err){
        res.send({code: 500, msg: err});
    }else{
        if(ret){
            var data = [];
            for(var r in ret){
                data.push({
                   route: r,
                   rtimes: ret[r],
                   qps: Math.floor(ret[r]/sec)
                });
            }

            res.send({code: 200, data: data});
        }else{
            res.send({code: 200, data: []});
        }
    }
});
```

`http://localhost:8000/api/qps?qpsDate=062116`表示查询06月21号下午16点-17点期间的qps情况。
