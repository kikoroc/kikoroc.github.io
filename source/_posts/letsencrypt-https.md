---
title: 使用letsencrypt实现https站点
date: 2016-09-23 10:17:03
tags: [https]
---

最近想把我的博客弄成https安全加密模式，其实不是为了什么安全，就是为了折腾下https~对于https证书的申请，推荐两个好用免费的[startSSL](https://www.startssl.com/)/[letsencrypt](https://letsencrypt.org/)。这里简单记录下使用letsencrypt的过程。

[letsencrypt的官网](https://letsencrypt.org/docs/client-options/)上提供了很多申请证书的工具，这里我选择了开源的python工具[acme-tiny](https://github.com/diafygi/acme-tiny)。

### 创建Let's Encrypt账号私钥

创建一个RSA私钥用于Let's Encrypt识别你的身份。

```bash
openssl genrsa 4096 > account.key
```

### 创建站点的CSR文件

CSR(Certificate Signing Request)就是证书签名请求文件，首先创建一个域名RSA私钥（不要使用上面的账号私钥）。

```bash
openssl genrsa 4096 > domain.key
```

接着就可以生成CSR文件了，在生成CSR文件的时候最好把带`www`和不带`www`两个域名都添加进去，如果有其他子域名再依次添加。

<!--more-->

```bash
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:kikoroc.com,DNS:www.kikoroc.com")) > domain.csr
```

### 配置站点验证文件

这个主要是用于Let's Encrypt来验证你对域名的所有权的，跟一些搜索引擎的站点认证原理差不多，在你的服务器上生成一个随机文件，再通过指定的域名访问，如果访问得到就说明你对域名有所有权。

首先创建验证文件目录:

```bash
mkdir -p /var/www/challenges/
```

接着修改nginx配置:

```bash
server {
    listen 80;
    server_name kikoroc.com www.kikoroc.com;

    location /.well-known/acme-challenge/ {
        alias /var/www/challenges/;
        try_files $uri =404;
    }

    ...the rest of your config
}
```

### 获取网站证书

先获取`acme-tiny.py`脚本。

```bash
wget https://raw.githubusercontent.com/diafygi/acme-tiny/master/acme_tiny.py
```

使用`acme-tiny`生成站点证书。

```bash
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /var/www/challenges/ > ./signed.crt
```

这个脚本就会验证上面生成的验证文件，验证通过就会生成站点的证书文件`signed.crt`。

### 在nginx中安装证书

在nginx中，还需要将Let's Encrypt的中间证书和网站证书合在一起：

```bash
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat signed.crt intermediate.pem > chained.pem
```

配置nginx ssl

```bash
server {
    listen 443;
    server_name kikoroc.com, www.kikoroc.com;

    ssl on;
    ssl_certificate /path/to/chained.pem;
    ssl_certificate_key /path/to/domain.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
    ssl_session_cache shared:SSL:50m;
    ssl_dhparam /path/to/server.dhparam;
    ssl_prefer_server_ciphers on;

    ...the rest of your config
}

server {
    listen 80;
    server_name kikoroc.com, www.kikoroc.com;

    location /.well-known/acme-challenge/ {
        alias /var/www/challenges/;
        try_files $uri =404;
    }

    ...the rest of your config
}
```

### 自动更新证书脚本

Let's Encrypt证书每次只有90天的有效期，可以通过脚本定期更新，so easy~

创建自动更新脚本`renew_cert.sh`:

```bash
#!/usr/bin/sh
python /path/to/acme_tiny.py --account-key /path/to/account.key --csr /path/to/domain.csr --acme-dir /var/www/challenges/ > /tmp/signed.crt || exit

wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem

cat /tmp/signed.crt intermediate.pem > /path/to/chained.pem

service nginx reload
```

添加crontab定时任务:

```bash
0 0 1 * * /path/to/renew_cert.sh 2>> /var/log/acme_tiny.log
```

### https引起的问题

一切配置好后以为万事大吉，chrome打开网站一看，虽然是https协议的，但是并没有期望中的小绿锁。打开chrome开发者工具发现问题所在：

```
Mixed Content: The page at 'https://kikoroc.com/sadsadsa' was loaded over HTTPS, but requested an insecure script 'http://www.qq.com/404/search_children.js'. This request has been blocked; the content must be served over HTTPS.
Mixed Content: The page at 'https://kikoroc.com/sadsadsa' was loaded over HTTPS, but requested an insecure image 'http://kikoroc.qiniudn.com/double.jpg'. This content should also be served over HTTPS.
GET https://push.zhanzhang.baidu.com/push.js net::ERR_INSECURE_RESPONSE
```

简单来说，就是网页中使用了一些第三方的http资源，这些被认为是有安全隐患的！

这些第三方资源又可以分为可控和不可控两类，比如第三方的图片等资源我们可以转存到自己的cdn服务商，对于不可控的第三方资源先尝试是否支持https，如果支持的话将引用的地方统一修改成引用地址，比如将`http://push.zhanzhang.baidu.com/push.js`修改为`//push.zhanzhang.baidu.com/push.js`；如果第三方不支持https那就暂时没什么办法了，比如腾讯公益404项目暂时就不支持https访问，只能去掉了。

上面说到转存到自己的cdn服务商，这里的前提条件是你的cdn服务支持https协议。本站采用的是qiniu的cdn，https服务现在是要收费的，不过对于我这类博客网站来说，价格还是能接受的。开通qiniu https域名服务后，将文章中的qiniu http域名替换成https的域名即可。

批量替换https域名：

```bash
find . -type f | xargs sed -i '' 's/http:\/\/kikoroc.qiniudn.com/https:\/\/odxth7737.qnssl.com/g'
```

对于duoshuo的头像和表情，推荐使用 https://github.com/rainwsy/duoshuo-https 解决。

期望的小绿锁终于出现了~~~

-EOF-
