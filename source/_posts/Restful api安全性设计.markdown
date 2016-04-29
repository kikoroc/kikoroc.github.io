title: Restful api安全性设计
date: 2014-11-11 11:11:11
tags: [restful, 安全, oauth2.0]
---

说到restful api的安全性，大家可能很快就想到OAuth2.0，的确，OAuth2.0是目前最主流和最安全的授权机制设计。但使用它的成本对于中小型应用来说还是比较高的。那么在不使用OAuth2.0的情况下如何设计restful api的安全性呢？
    
业内还真有不使用OAuth2.0来做api鉴权的大公司——大名鼎鼎的`Amazon web service`。且看amazon是如何来做的，查阅amazon相关的文档我们大概知道其中设计的原理：为client生成public key和private key对，其中public key当然任何人都可以知道，或者可以叫做appid，用于server来区分是哪个client的；private key（或者可以叫做app secret）则只能是server和client知道，不可泄漏给第三方的任何人；client在做request的时候，将request的参数和取值整合成一个string，然后对这个string使用private key计算签名值（HMAC-SHA1或者SHA256之类），然后将这个签名值连同请求参数一起发送给server，server收到request之后按照同样的逻辑整合string，用同样的算法计算签名值，最后比较server生成的签名和client生成的签名是否相同，相同则client可信，继续执行业务逻辑，不相同则拒绝访问。

amazon的第一版方案是这么做的：


> 1.Split the query string based on '&' and '=' characters into a series of key-value pairs.

> 2.Sort the pairs based on the keys.

> 3.Append the keys and values together, in order, to construct one big string (key1 + value1 + key2 + value2 + ... ).

> 4.Sign that string using HMAC-SHA1 and your secret access key.

<!--more-->

技术是不断发展的，很快我们就发现amazon的第一版方案并不是安全的，[AWS signature version 1 is insecure](http://www.daemonology.net/blog/2008-12.html)，大概就是说这个方案存在碰撞的可能性，很简单，举个例子，`key=value`的签名和`ke=yvalue`的签名是一样的，很明显吧。所以接着有了amazon的第二版方案[Making Secure Requests to Amazon Web Services](http://aws.amazon.com/articles/1928)。感兴趣可以看看第二版方案改进的地方。

那么，我们的方案就是基于amazon的思路来的：


> 1.[Client]整合请求参数key，将key排序，连接key和value以及=和&等得到string；

> 2.[Client]我们还需要在1中的string上加上请求的URI以及Request method，得到新的string，这样我们可以避免可能有多个api的参数列表是一样的导致不安全，比如/api/update和/api/delete有同样的参数，这样算得的签名是一样的，但执行结果就完全不同；

> 3.[Client]为了防止重放攻击，我们还需要在2中的string中增加当前请求时间戳的字段，这样server收到请求后可以计算时间间隔，超过一定时间范围的请求将拒绝访问，为了避免时区的问题，我们可以直接使用[UTC Time](http://www.thebuzzmedia.com/understanding-the-unix-epoch-in-java-time-zones-and-utc/)；

> 4.[Client]将3中得到的string url encode后使用private key计算签名（HMAC-SHA1、HMAC-SHA256、SHA256，建议HMAC-SHA256）；

> 5.[Client]将请求参数连同签名一起发送给server；

> 6.[Server]收到请求后，首先获取时间戳判断请求是否过期；

> 7.[Server]根据public key查询到对应的private key；

> 8.[Server]按照client一样的逻辑整合请求string；

> 9.[Server]server端计算签名；

> 10.[Server]比较server计算的签名和客户端请求的签名，如果相同则client可信，否则拒绝访问。


最后，拆屋卖瓦都要配上`SSL`！

上文说到的安全性设计在理想的情况下还是不错的，这里说的理想情况是指private key在server和client都绝对安全的情况，但事实上反编译client后很容易就拿到了private key，一旦private key泄漏，安全瞬间就成浮云了-.-这个问题本人尚未探索到解决方案，慢慢来吧~