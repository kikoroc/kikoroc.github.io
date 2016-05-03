---
title: Redis最佳实践  
date: 2016-05-03 10:16:55  
tags: [redis,best practice]  
---


> [redis](http://redis.io/)是一款开源的内存数据存储系统，可以用作数据库、缓存甚至是消息中间件（pub/sub）来使用。与memcache相比，redis支持更多的数据结构，比如string,hash,list,set,bit map,sorted set甚至是geo等等，基本覆盖了日常开发中使用到的数据结构。而且redis十分高效，当然是建立在合理使用的前提下。由于redis单线程的设计特性，任何一条阻塞的命令都会引起redis整个实例的阻塞，所以在使用过程中，需要尽量避免过多使用时间复杂度高的命令（特别是高并发的环境下），一般我们可以通过查看redis command的时间复杂度，但实际使用情况中还是会遇到不少坑，本文主要记录工作中遇到的redis问题，持续更新~~~

<!--more-->


## 找出引起redis慢的原因

### slowlog get

通常我们可以通过redis慢日志来找到引起redis慢的命令，用法为`slowlog get 10`来查看最慢的10条命令，然后针对性的进行优化。慢日志可以通过redis.conf或者运行时通过：
```
CONFIG SET slowlog-log-slower-than 5000  
CONFIG SET slowlog-max-len 25
```

来设置slowlog参数，其中`slowlog-log-slower-than`表示执行时间超过该值（单位毫秒）的命令记为慢查询，`slowlog-max-len`可以设置记录的最大条数。可以通过`slowlog reset`命令来重置慢日志记录。

### info commandstats

`info commandstats`命令会告诉你整个redis执行了哪些命令、分别执行了多少次、总计耗时、平均每次耗时等信息，同时可以通过`config resetstat`命令来重置统计。

### client list

`redis-cli -h localhost -p 6379 client list | grep -v "omem=0"`这条命令在排查redis慢的时候绝对是神技。一般阻塞的命令都会导致omem不断升高，这条命令能快速找到引起阻塞的命令，返回的数据:
```
id=1212 addr=10.10.10.10:34234 fd=11 name= age=3242 idle=1 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=232432 omem=23132432 events=rw cmds=keys *
```
引起阻塞的具体命令，甚至发起命令的机器ip都能查到。

## scan instead of keys *

在key数量较少的情况下，我们可以偷懒直接使用`keys *`来快速查看模式匹配的key列表，但是一旦key数量上升，或者在高并发的环境下，`keys *`会带来整个系统的阻塞。因为`keys`命令时间复杂度是：

> **Time complexity**:O(N) with N being the number of keys in the database, under the assumption that the key names in the database and the given pattern have limited length.

直接跟database中key数量相关的。替代方案是使用`scan`命令，首先看看`scan`的时间复杂度：

> **Time complexity**:O(1) for every call. O(N) for a complete iteration, including enough command calls for the cursor to return back to 0. N is the number of elements inside the collection.

虽然scan不能一次性返回所有匹配的key，但是scan提供cursor机制来遍历整个database，最重要的是每次scan操作的时间复杂度是O(1)的，因此只需要多次scan即可得到所有匹配的key。类似的命令还有`sscan`,`hscan`,`zscan`等分别用于增量迭代set,hash,sorted set等集合元素。

```
var scan = function(offset){
  (function(offset){
    redis.scan([offset, 'match', 'key_pattern_*', 'count', 1000], function(err, ret){
      if(!!err){
        console.error(err);
        process.exit(-1);
      }
      if(Number(ret[0] === 0)){
        console.log('scan finished');
        process.exit(0);
      }
      // matched keys in ret[1]
      // deal with keys
      // scan again.
      scan(Number(ret[0]));
    });
  })(offset);
};

scan(0);
```

## 使用redis slave

同使用其他数据库一样，读写分离总是能极大的提升系统在高并发情况下的性能，redis也不例外。

### 一主多从

通常一主多从可以用来实现读写分离，而且能间接的保证数据安全性（master挂了slave还有数据），所以通常比较好的部署结构是M-S-S，即在slave下部署slave而不是全部部署为master的slave，这样master挂了可以将一级slave快速切换为master使用。

### 多写

多写一般用于写操作很频繁的情况，这时一般需要业务上进行额外的处理，或者更好的办法是增加redis proxy类的中间件来对业务隔离多写的复杂性。

## DEL操作隐藏的问题

之前一直以为`del`操作不会有性能问题，直到我的膝盖中了一箭...
因为只记得`del`对于string和hash的时间复杂度是O(1)的，但是对于list,set,sorted set等居然是O(N)的，所以当你准备使用del来删除一个有百万级数据的集合，那你就准备阻塞吧...  

>**Time complexity**:O(N) where N is the number of keys that will be removed. When a key to remove holds a value other than a string, the individual complexity for this key is O(M) where M is the number of elements in the list, set, sorted set or hash. Removing a single key that holds a string value is O(1).

我们的方案是：不直接删除这种大的集合，而是将他们重命名(确认是O(1)的，不用担心:D)，然后后台跑一个删除进程慢慢删。。。   
首先，将程序中del大集合修改为rename：  

```
//cmds.push(['del', 'bigset']);  
cmds.push(['rename', 'bigset', 'gc:bigset']);
```

接下来部署删除函数：  

```
var delHugeSet = function(key, cb){
    redis.scard(key, function(err, size){
        if(size>500){
            redis.srandmember(key, 500, function(err, ids){
            	//分批慢慢删
                redis.srem(key, ids, function(err, ret){
                    delHugeSet(key, cb);
                });
            });
        }else{
        	 //数据量不大，直接del
            redis.del(key, function(err, ret){
                cb();
            });
        }
    });
};
```
