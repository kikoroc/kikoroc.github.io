---
title: 移动app好友关系设计以及数据同步策略
date: 2016-05-27 16:05:18
tags: [mobile]
---

带有社交和IM属性的APP中，离不开好友关系的功能设计，从技术角度来看，主要包括**好友关系存储方案**和**好友数据同步方案**。这里简单介绍下我目前开发的app中好友关系及数据同步的方案设计思路。

## 背景

在我们的app设计中，好友关系是一对一双向的，即一方申请加另一方好友后，需要另一方同意后双方才建立好友关系，任一方删除好友后，那么双方互为非好友关系；  
在开始的设计中，用户登录后，每次以分页的方式拉取好友列表，这样存在比较多的问题：  
>1.每次都会发起拉取好友列表的请求，浪费流量的同时增加服务器压力；  
2.客户端本地无法判断某人是否是自己的好友，需要每次去服务端查询；  
3.打开玩家详情的时候，每次都需要去服务端更新数据；  

可以看到这种方案很低效而且成本很高，引入好友数据同步方案之后可以大大改善这种问题，客户端拉取好友数据到本地后缓存起来，直接在本地就可以判断好友关系、查询好友数据等，然后通过增量同步的方式与服务端保持数据同步。

<!--more-->

## 好友关系设计

基于我们的产品需求，我们的好友关系采用单条记录保存，即一条好友关系就一条数据库表记录。

好友关系表设计：

```sql
-- 好友关系表
create table if not exists `tb_relation` (
    `id` bigint(20) unsigned auto_increment not null comment '自增id',
    `user1` bigint(20) unsigned not null comment '用户(主动)id',
    `user2` bigint(20) unsigned not null comment '用户(被动)id',
    `relation` tinyint(1) unsigned not null comment '关系，0黑名单1好友2解除好友',
    `dealed` tinyint(1) unsigned not null comment '是否处理0否1是',
    `updateAt` bigint(20) not null comment '关系变更时间戳',
    `createAt` bigint(20) not null comment '关系发生时间戳',
    primary key (`id`),
    unique key `unq_user1_user2_rela` (`user1`, `user2`),
    index `idx_user2` (`user2`),
    index `idx_updateAt` (`updateAt`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;
```

其中user1是主动发起好友请求的用户id，整个表都是int数据表空间占用很小而且建立适当的索引之后，即使表数据量很大的情况下，查询效率依然很高。我们当前单表数据量6000w+，单mysql实例仍然可以保持4k qps的查询。

* 发起好友请求

```sql
insert ignore into tb_relation values (1,2,2,0,1443242353243,1443242353243);
```

如果已经存在记录而且当前不是好友关系则：

```sql
update tb_relation set user1=?,user2=?,relation=2,dealed=0,updateAt=? where id = ?;
```

重新置为未处理的好友申请即可（可能存在user1,user2对调的情况，不过还是以user1为主动用户）。

* 好友申请列表

```sql
select id,user1 from tb_relation where user2=? and dealed=0;
```

user2被动，所以where条件是user2并且dealed为0。（可能关联账号表或者从缓存中取出申请者的信息，这里略了）。

* 处理好友请求

```sql
update tb_relation set relation=?,dealed=1,updateAt=? where id = ?;
```

其中relation根据同意和拒绝分别取值1和2。

* 删除好友

删除好友并不会删除数据库记录，而是更新relation字段。

```sql
update tb_relation set relation=2,updateAt=? where id = ?;
```

## 好友关系同步

![friend data sync](http://kikoroc.qiniudn.com/friend_data_sync.jpg)
本同步方案的关键在于记录updateAt。

* 全量同步fullSync

客户端登录后，如果本地没有账号的好友数据（第一次登录或者数据被清理了）或者本地的同步标记lastSyncTime不存在，那么这时客户端需要请求服务端进行好友关系数据的全量同步，全量同步的意思就是服务端查询全部的好友数据信息返回给客户端，并附带当前操作的时间戳作为客户端本次同步的同步标记lastSyncTime；

客户端收到全量同步的数据之后，保存在本地并记录同步标记lastSyncTime，之后客户端在线的情况下加好友或者被删等通过服务器推送直接实时更新本地好友关系数据即可。

* 差异同步diffSync

客户端下次登录之后，如果本地有好友数据和同步标记lastSyncTime（没有的话走fullSync），只需请求服务端进行差异更新即可，差异更新的时候带上本地同步标记lastSyncTime，由服务端计算lastSyncTime和好友数据中updateAt之间差异的数据返回，并附带当前操作的时间戳作为客户端本次同步的同步标记lastSyncTime；

客户端收到差异更新的数据之后，更新本地好友数据并更新lastSyncTime。
ps：差异更新返回的数据中relation=1表示是新的好友，relation=2表示该好友关系已被删除，客户端做对应的逻辑即可。

有了同步机制之后，客户端只需要每次登陆后差异更新即可，这样数据量和对服务器的压力都会得到很大的减少，而且是否好友的判断可以直接在本地判断，还可以直接在本地做好友搜索，很方便。

哦，还有一点，本地好友的信息可能不是最新的，因为上面的同步方案中并没有感知到好友基本信息的修改，比如某个好友修改了自己的昵称或者头像，你的好友列表中看到的可能还是旧的昵称和头像，但其实用户并不关心其实时性，他们的信息变动完全可以在查看其详细信息时再更新（微信的策略），当然如果非要实时不是没办法，比如可以增加好友信息变更的推送，好友信息修改之后修改tb_relation的updateAt，让其好友通过差异更新的方式得到最新的信息，你需要权衡下这么做的成本和效果。因为*很多时候并没有什么最佳实践，只有最合适的实践*。
