title: 新浪微博WB2.login授权登录获取帐号信息以及Access_token的解决办法
date: 2014-09-20 16:53:23
tags: [javascript, weibo]
---

### 问题1.获取帐号信息

最近项目中要使用到新浪微博登录，客户端的已经搞定，在网页端也需要使用微博登录功能，查了下[微博的文档](http://open.weibo.com/)，说实话写的不太好，很多地方没有清楚而且页内的链接跳转也略混乱，不过api的设计倒是不错的。在使用过程中发现，WB2.login是没有返回值的，这时候想要拿到登录用户的信息仅通过WB2.login接口是没办法做到的。

### 思路

继续查阅weibo-js的文档，发现了`/account/get_uid.json`（OAuth授权之后获取用户UID）和`/users/show.json`（获取用户信息）这个两个api，所以大概的思路就有了：先请求`/account/get_uid.json`获取授权用户的uid，再在请求uid的回调中调用`/users/show.json`获取用户数据。虽然这个做法比较绕，但暂时也没发现其他更好的办法，大家可以互相交流：）

<!--more-->

**代码实现**

```
;(function($){
	/**
	 * 登录新浪微博，并获取用户信息
	 */
	var loginWb = function(){
		WB2.login(function(){
		    //检查登录是否成功
			if(WB2.checkLogin()){
				getWbUserData(function(data){
					alert("登录成功！");
					console.log(data);
				});
			}else{
				alert("登录新浪微博失败！");
			}
		});
	};
	
	/**
	 * 退出新浪微博
	 */
	var logoutWb = function(){
		WB2.logout(function(){
			if(WB2.checkLogin()){
				alert("退出新浪微博失败！");
			}else{
				alert("退出新浪微博");
			}
		});
	};
	
	/**
	 * 获取微博用户数据
	 */
	var getWbUserData = function(callback){
		//调用api
		WB2.anyWhere(function(W){
			W.parseCMD("/account/get_uid.json",function(ret,status){
				if(status){
					getData(W,ret.uid);
				}else{
					console.log("/account/get_uid.json 授权失败或者错误");
				}
			},{},{
				method:"get",
				cache_time:30
			});
		});
		function getData(W,id){
			W.parseCMD("/users/show.json",function(ret,status){
				if(status){
					callback(ret);
				}else{
					console.log("/users/show.json error");
				}
			},{
				uid:id
			},{
				method:"get"
			});
		}
	};
}(Zepto));
```

### 问题2.获取access_token

授权登录，拿到帐号数据后发现，想拿到access token也是个事！其实access token在登录的时候就已经生成了的，但是weibo-js并没有提供一个简便的方式去获取access token，至少api文档里面没有提到，获取access token只有一个api `oauth2/accesstoken`，但是看看这个api的说明：

**请求参数**

| 	            |必选	|类型及范围	|说明                               |
|:----------    |:-----:|:---------:|:----------------------------------|
|client_id      |true	|string	    |申请应用时分配的AppKey。           |
|client_secret  |true	|string	    |申请应用时分配的AppSecret。        |
|grant_type     |true	|string	    |请求的类型，填写authorization code |

**grant type为authorization code时**

|       |必选	|类型及范围|	说明|
|:----- |:----:|:---:|:-----|
|code	|true|	string|	调用authorize获得的code值。|
|redirect_uri|	true|	string|	回调地址，需需与注册应用里的回调地址一致。|

看到这些参数就知道这不是我们想要的api：（
根据文档的说明，在开发的时候可以设置`debug=true`来的打印调试信息。

```
<script src="http://tjs.sjs.sinajs.cn/open/api/js/wb.js?appkey={YOUR_APPKEY}&debug=true" type="text/javascript" charset="utf-8"></script>
```

请求授权登录后在console里面发现：

![Alt text](http://kikoroc.qiniudn.com/1411210569711.png)

我们想要的access token被log出来了，显然这是wb.js的逻辑，继续查看wb.js的源码，找到`console.log`的地方。

![Alt text](http://kikoroc.qiniudn.com/1411210702079.png)

哈哈，被log的data数据接着被`Cookie.save(data)`了。
继续看cookie信息，果然是有一个key为weibojs接一串数字，value则是access token等信息。而且weibojs后面那个数字正是我们申请到的app key，接下来就好做了，拿到这个cookie数据，解析出access token即可。

![Alt text](http://kikoroc.qiniudn.com/1411210778626.png)

我这里是用到zepto.js库，zepto没有自带cookie相关api的，找到[zepto的cookie扩展](https://github.com/kennx/Zepto-Cookie)，获取access token的代码：

```
var getAccessToken = function(){
	var cookie = $.fn.cookie("weibojs_{your_appkey}");
	return cookie.split("&")[0].split("=")[1];
};
```

搞定收工！

