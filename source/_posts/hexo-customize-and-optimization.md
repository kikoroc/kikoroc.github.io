---
title: hexo定制and优化
date: 2016-05-05 15:36:13
tags: [hexo, nodejs]
---

## 前言
折腾hexo有几天时间了，配合[NexT](https://github.com/iissnan/hexo-theme-next)主题，大爱！虽然我写博客主要是用来记录自己的一些知识积累，并不强求有多少人来看，hexo本身已经很优秀了，但有些地方对于强迫症来说还是忍不住想要定制和优化，这样才能让自己更爽~~而且折腾的过程中，也可以学到不少新的知识。

## 静态资源优化
主要是压缩html,css,js等等静态资源，可以适当减少请求的数据量，主要用到`gulp`和一些相关的插件来实现，直接列出我的dependencies.
```
"dependencies": {
  "gulp": "^3.9.1",
  "gulp-htmlclean": "^2.7.6",
  "gulp-htmlmin": "^1.3.0",
  "gulp-imagemin": "^2.4.0",
  "gulp-minify-css": "^1.2.4",
  "gulp-uglify": "^1.5.3",
  "hexo": "^3.2.0",
  "hexo-deployer-git": "^0.1.0",
  "hexo-generator-archive": "^0.1.4",
  "hexo-generator-category": "^0.1.3",
  "hexo-generator-index": "^0.2.0",
  "hexo-generator-tag": "^0.2.0",
  "hexo-generator-sitemap": "^1.1.2",
  "hexo-generator-baidu-sitemap": "^0.1.2",
  "hexo-renderer-ejs": "^0.2.0",
  "hexo-renderer-marked": "^0.2.10",
  "hexo-renderer-stylus": "^0.3.1",
  "hexo-server": "^0.2.0"
}
```
其中`gulp`相关的就是用来压缩静态资源用的，接着还要安装`gulp`工具：
<!--more-->
```
$ npm install gulp -g
```
然后添加`gulpfile.js`到根目录。
```
var gulp = require('gulp');
var minifycss = require('gulp-minify-css');
var uglify = require('gulp-uglify');
var htmlmin = require('gulp-htmlmin');
var htmlclean = require('gulp-htmlclean');
// 压缩css文件
gulp.task('minify-css', function() {
  return gulp.src('./public/**/*.css')
  .pipe(minifycss())
  .pipe(gulp.dest('./public'));
});
// 压缩html文件
gulp.task('minify-html', function() {
  return gulp.src('./public/**/*.html')
  .pipe(htmlclean())
  .pipe(htmlmin({
    removeComments: true,
    minifyJS: true,
    minifyCSS: true,
    minifyURLs: true,
  }))
  .pipe(gulp.dest('./public'))
});
// 压缩js文件
gulp.task('minify-js', function() {
  return gulp.src('./public/**/*.js')
  .pipe(uglify())
  .pipe(gulp.dest('./public'));
});
// 默认任务
gulp.task('default', [
  'minify-html','minify-css','minify-js'
]);

```
以后generate的时候跟上gulp命令即可压缩静态资源。
```
$ hexo g
$ gulp
$ hexo d
```

## 生成sitemap文件
生成sitemap文件然后提交给搜索引擎，对于SEO很有帮助，hexo有相关的sitemap插件。
```
"hexo-generator-sitemap": "^1.1.2",
"hexo-generator-baidu-sitemap": "^0.1.2",
```
安装这俩插件后，以后每次`hexo g`都会生成sitemap.xml和baidusitemap.xml文件并自动帮你放到public目录。很省心~

## 修改permalink
hexo默认的permalink配置是：
```
permalink: :year/:month/:day/:title/
```
这样生成的访问链接都是类似`http://yoursite.com/2016/05/06/your_title/`这样的，有点动态网站的风格，不过这样可能对于搜索引擎不是太友好，我们可以把permalink修改成.html结尾的：
```
permalink: :year/:month/:day/:title.html
```
还有一点需要注意的是，title最好是英文的，这样permalink也是不带中文的，但是markdown文件中的title你还是可以按自己的文章标题写成中文的。

## 其他小的修改点
这些都是看个人爱好了，hexo还是很方便自己自定义的。
### 站点头像改成圆形
在`themes/next/source/css/_common/components/sidebar/sidebar-author.styl`中`.site-author-image`定义中增加：
```
border-radius: 50%;
-webkit-border-radius: 50%;
-moz-border-radius: 50%;
```
### 去掉站点链接前面的小圆点
个人不太喜欢站点链接前面的小圆点，去掉`themes/next/source/css/_common/components/sidebar/sidebar-author-links.styl`中`a::before`的定义即可。

### 小技巧
有其他想修改的小地方，都可以打开浏览器的开发者工具，找到对应地方的css定义，然后在sublime或者atom中`Search In Directory`找到对应css的源码，修改即可。

## 解决github pages屏蔽百度爬虫的问题
嗯，是的，github pages屏蔽了百度爬虫(屏蔽了百度爬虫的UA`Mozilla/5.0 (compatible; Baiduspider/2.0; +http://www.baidu.com/search/spider.html)`)。这样你的hexo博客很难被百度收录，从百度站长工具抓取诊断中就可以发现爬取都是403 forbidden。
```
HTTP/1.0 403 Forbidden
Cache-Control: no-cache
Connection: close
Content-Type: text/html
```
本来也无所谓，但是从技术的角度来看，解决这种问题肯定很有快感~~
目前主要有两种解决方案：`CDN回源`和`为百度spider指定特殊线路`。两种方案我都试过，最终选择了后者。
### CDN回源
hexo博客都是静态文件，很适合使用CDN回源，这样不仅能提高访问速度还能跳过github对百度爬虫的屏蔽。
回源的意思是用户请求hexo之后CDN会抓取hexo内容，这样其他用户就不用访问github pages了而是直接从CDN就近的节点拉取数据，访问速度更快。但是这里有个问题，比如百度爬虫所在的区域CDN节点还没有缓存hexo数据，这样百度爬虫会穿透CDN直接请求到github pages，然后后果就是github pages返回403 forbidden。所以CDN回源的方式仅适用于博客访客量比较大的情况（这样访问会在百度爬虫之前让CDN缓存hexo数据，当然也只是比较大的概率）。
不过如果无视百度爬虫，给hexo加上CDN回源也是很不错的，至少能提高访问速度。推荐使用[又拍云](https://www.upyun.com/index.html)。大概的配置步骤：
#### 配置服务
![配置upyun服务](https://odxth7737.qnssl.com/upyun_cdn_1.png)
#### 添加CNAME解析
```
CNAME @ kikoroc.b0.aicdn.com
```
同时去掉到github pages ip的A类解析。
### 为百度spider指定特殊线路
除了github pages能托管hexo之外，还有其他的选择，比如gitcafe pages(已经和coding.net合并)。传送门：[gitcafe pages配置](https://coding.net/help/doc/pages/index.html)，配置还是挺简单的。而且gitcafe没有屏蔽百度爬虫。但是gitcafe pages的服务器目前还不够github给力（偶尔出现一些无法打开的情况），所以有了新的思路：
#### hexo deploy同时部署到github和coding.net
```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  #repo: https://github.com/kikoroc/kikoroc.github.io.git
  #branch: master
  repo: git@git.coding.net:/kikoroc/kikoroc
  branch: coding-pages
```
后面有时间可以写个脚本直接同时部署github和gitcafe。
#### 正常用户访问github pages，百度爬虫访问gitcafe
现在的DNS解析服务一般都可以指定特定的线路，比如指定百度爬虫的线路解析到gitcafe。
![dnspod设置](https://odxth7737.qnssl.com/dnspod1.png)
gitcafe pages绑定域名到kikoroc.com。

-EOF-
