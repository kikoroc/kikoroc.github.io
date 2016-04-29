title: 第三方帐号OAuth授权登录安全问题
date: 2014-11-19 20:21:21
tags: [oauth2.0, 安全]
---


在我们做产品的时候，选择第三方帐号登录的设计，对于用户而言的确是很方便，可以省去用户注册的流程，用户也不用费心去记多一个帐号和密码，可以用QQ、微博等帐号玩遍天下，而且使用第三方帐号系统可以更好的与第三方系统集成，更方便的与好友分享和互动。
    
### OAuth优点

OAuth不会使第三方应用或者网站接触到用户的敏感数据（用户名密码等），而是通过授权后的access_token来实现用户数据交互，在http中传输的不再是用户的敏感数据而是access_token，所以即使数据包被拦截，也难以拿到用户敏感数据。这是OAuth最大的优点。

### OAuth授权的安全隐患

在最近的项目中，我们使用到了微博、qq、微信等第三方帐号登录功能，在接入这些第三方帐号的时候，发现一些值得思考的问题。以新浪微博的OAuth为例，在网页中接入微博登录主要使用weibo的[jssdk](http://tjs.sjs.sinajs.cn/open/api/js/wb.js)，分析jssdk的源码我们发现，在用户通过OAuth授权登录后，jssdk将授权后weibo返回的access_token是存入cookie的。

<!--more-->

```
var Cookie = {
		load: function() {
			if (!$.core.obj.isEmpty(WB2.oauthData)) {
				return WB2.oauthData
			} else {
				var _cookie = CookieUtil.get("weibojs_" + WB2._config.appkey);
				_cookie = unescape(_cookie);
				var oCookie = queryToJson(_cookie);
				return oCookie
			}
		},
		save: function(oCookie) {
			WB2.oauthData = oCookie;
			var _cookie = "access_token=" + (oCookie.access_token || "") + "&refresh_token=" + (oCookie.refresh_token || "") + "&expires_in=" + (oCookie.expires_in || 0) + "&uid=" + (oCookie.uid || "") + "&status=" + (oCookie.status || USER_STATUS || -1);
			CookieUtil.set("weibojs_" + WB2._config.appkey, _cookie, {
				path: "/",
				domain: document.domain
			})
		},
		del: function() {
			WB2.oauthData = {};
			CookieUtil.remove("weibojs_" + WB2._config.appkey, {
				path: "/",
				domain: document.domain
			})
		}
	};
```

jssdk将授权后得到的access_token、refresh_token、expires_in、uid、status等数据写入到以weibojs_{appkey}命名的cookie中，通过chrome的开发者工具，确实可以看到如下信息：

![jssdk cookie](http://kikoroc.qiniudn.com/jssdkcookie.jpg)

当然，这里的安全已经不是微博能控制的了，因为这已经是在第三方站点了。设想下，如果这第三方网站存在XSS漏洞，黑客可以很容易的通过XSS漏洞拿到cookie数据，也就拿到用户的access_token，拿到这个访问令牌就能干很多坏事了，因为微博的api调用都是通过这个令牌来完成的。比如关注某个用户的api为：

![create friendship](http://kikoroc.qiniudn.com/createfri.jpg)

有了用户的access_token就能很容易的让用户关注任何人了！还有更多的api来做坏事！
那自己的微博做了下测试，取消关注好友：

![destory friendship](http://kikoroc.qiniudn.com/destoryfri.jpg)

一个简单命令行就能取消关注好友！

### OAuth服务提供者

在大多数情况下，出问题的环节往往不是技术，而是使用技术背后的人。

- 规范部署OAuth，强制SSL
- 平台对于第三方应用严格仔细审核

### 第三方应用开发者

- 自律！do no evil.
- 安全意识和措施

### 广大用户

- 防止假冒OAuth登录页面钓鱼
- 授权前三思
- 及时清理不必要的授权
- 授权后观察其来源，“未审核应用”这类建议先取消授权，等其通过审核再说

对于我们第三方应用的开发者而言，除了自律不作恶，我们还要提高我们的安全意识，为我们的程序增强安全防范措施，为用户提供安全放心的产品。ps：接下来研究下xss及其防范手段。