---
title: [分享一个将pomelo config解析为pm2 config的脚本]
date: 2016-06-15 16:54:37
tags: [pomelo, nodejs]
---

pomelo原生自带的pomelo start等命令可以很好的在开发过程中使用，但是到产品部署阶段就显得有些力不从心。在pomelo集群部署方面，推荐使用[pm2](http://pm2.keymetrics.io/docs/usage/quick-start/)来运维部署pomelo服务器组。用过才知道真的比pomelo start等命令爽快很多，墙裂推荐。

本文主要分享一个将pomelo config解析为pm2 config的脚本。

**parse_pomelo_to_pm2_config.js**

```javascript
/**
 * 将pomelo默认配置servers.json转为pm2配置
 * Created by kikoroc on 16/4/26.
 */
var env = 'production';
var servers = require('../servers.json')[env];
var master = require('../master.json')[env];
var masterha = require('../masterha.json').masterha;
var fs = require('fs');

/**
 * 一台服务器可能有多个网络类型的ip地址（电信移动联通局域网等），这里主要配置将服务器多个ip解析为同一个ip
 * 这样就可以将同服务器的pomelo config配置到一个pm2 config json文件中。
 */
var _SERVER_IPMAP = {
    "your_server_telecom_ip": "your_server_telecom_ip",
    "mobile_ip": "your_server_telecom_ip",
    "unicom_ip": "your_server_telecom_ip",
    "local_ip": "your_server_telecom_ip"
};

var _RESULT = {};

for(var serverType in servers){
    servers[serverType].forEach(function(conf){
        var process = {
            "name": "",
            "cwd": "/data/xxx/game-server/",
            "script": "app.js",
            "args": null,
            "exec_mode": "fork",
            "watch": false,
            "env": {
                "NODE_ENV": "production"
            },
            "merge_logs": true,
            "error_file": "",
            "out_file": "",
            "autorestart": true,
            "max_memory_restart": "1G"
        };
        process.name = conf.id;
        process.error_file = "/data/xxx/game-server/logs/"+conf.id+"_error.log";
        process.out_file = "/data/xxx/game-server/logs/"+conf.id+"_normal.log";
        var args = [];
        args.push("env=production");
        args.push("serverType="+serverType);
        for(var k in conf){
            args.push(k+"="+conf[k]);
        }
        process.args = args;

        var destHost = _SERVER_IPMAP[conf.host];
        if(_RESULT.hasOwnProperty(destHost)){
            _RESULT[destHost].apps.push(process);
        }else{
            _RESULT[destHost] = {};
            _RESULT[destHost]['apps'] = [];
            _RESULT[destHost].apps.push(process);
        }
    });
}

masterha.forEach(function(conf){
    fs.writeFileSync('./out/masterha@'+_SERVER_IPMAP[conf.host]+'.json', JSON.stringify({
        "apps": [{
            "name": conf.id,
            "cwd": "/data/xxx/game-server/",
            "script": "app.js",
            "args": [
                "serverType=master",
                "id="+conf.id,
                "host="+conf.host,
                "port="+conf.port,
                "env=production",
                "mode=stand-alone",
                "masterha=true"
            ],
            "exec_mode": "fork",
            "watch": false,
            "env": {
                "NODE_ENV":"production"
            },
            "merge_logs": true,
            "error_file": "/data/xxx/game-server/logs/"+conf.id+"_error.log",
            "out_file": "/data/xxx/game-server/logs/"+conf.id+"_normal.log",
            "autorestart": true,
            "max_memory_restart": "1G"
        }]
    }, null, 2), {
        flag: 'w'
    });
});

//write master config
fs.writeFileSync('./out/master@'+_SERVER_IPMAP[master.host]+'.json', JSON.stringify({
    "apps": [{
        "name": master.id,
        "cwd": "/data/xxx/game-server/",
        "script": "app.js",
        "args": [
            "serverType=master",
            "id="+master.id,
            "host="+master.host,
            "port="+master.port,
            "env=production",
            "mode=stand-alone"
        ],
        "exec_mode": "fork",
        "watch": false,
        "env": {
            "NODE_ENV":"production"
        },
        "merge_logs": true,
        "error_file": "/data/xxx/game-server/logs/"+master.id+"_error.log",
        "out_file": "/data/xxx/game-server/logs/"+master.id+"_normal.log",
        "autorestart": true,
        "max_memory_restart": "1G"
    }]
}, null, 2), {
    flag: 'w'
});

for(var host in _RESULT){
    fs.writeFileSync('./out/'+host+'.json', JSON.stringify(_RESULT[host], null, 2), {
        flag: 'w'
    });
}
```

<!--more-->
运行

```bash
node parse_pomelo_to_pm2_config.js
```

pm2的json config文件就会生成在`./out/`目录下。然后分发到对应的服务器执行`pm2 start xxx.json`即可。

在实际运维过程中，可能需要重启某些serverType的服务器进程，这里再分享一个根据serverType生成对应服务器进程重启的脚本。

**gen_restart_shell_by_serverType.js**

```javascript
/**
 * 生成重启业务服务进程的pm2代码
 * Created by kikoroc on 16/4/27.
 */
var serverType = 'chat';
var servers = require('../../servers.json')['production'][serverType];

if(servers && servers.length > 0){
    servers.forEach(function(svr){
        console.log('ssh '+svr.host+' /usr/local/bin/pm2 restart '+svr.id);
    });
}else{
    console.error('no server of '+serverType);
}
```

运行

```bash
node gen_restart_shell_by_serverType.js > restart_chat.sh
bash restart_chat.sh
```

即可完成chat servers的重启。：）
