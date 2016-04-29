title: 使用HTML5 File API和Canvas实现图片压缩、旋转、上传
date: 2014-11-10 19:23:32
tags: [javascript, html5]
---



差不多一个多月没写博客了，不过也没闲着，完成了 **几件大事** ：结婚、户口、驾照、买车。感觉写代码更有劲了~哈哈...
    
回到今天的主题，最近项目上有一个拍照的需求，对于客户端当然是个小问题，但是产品需要在网页版的页面上同样要实现跟客户端一样的体验！让用户可以在网页上拍照或者选取本地照片上传到服务器。

看到这个需求有点蒙，首先还不确定网页如何调用系统相机，选本地照片的话弄个`<input type="file">`应该就ok，其次手机拍一张照片都是几兆几兆的，如果不压缩一下图片，在这蛋疼的网络环境下，基本是没办法传到服务器的，网页上的环境也就那样，怎么做图片压缩呢？

## 网页拍摄照片&选取本地照片

这个看起来很麻烦，其实很简单，网页上可以调用系统原生的照片拍摄和选取操作。直接上代码：

```
<input type="file" id="photo" accept="image/*" capture="camera">
```

其中，`accept`和`capture`是[HTML5的API](http://www.w3.org/TR/html-media-capture/)，这样在网页上点击这个`input`之后会弹出拍摄和选取本地照片的窗口，和native app一样。

## 照片读取、压缩

上面说到，如果不对照片进行处理的话，一张照片就是好几兆，在2g网络下基本不可能成功上传到服务器，所以肯定需要对照片进行压缩了。我们可以使用HTML5的File API配合Canvas绘图实现照片的压缩，代码：

<!--more-->

#### 1.读取照片

```
//绑定input change事件
$("#photo").unbind("change").on("change",function(){
    var file = this.files[0];
    if(file){
        //验证图片文件类型
        if(file.type && !/image/i.test(file.type)){
            return false;
        }
        var reader = new FileReader();
        reader.onload = function(e){
            //readAsDataURL后执行onload，进入图片压缩逻辑
            //e.target.result得到的就是图片文件的base64 string
            render(e.target.result);  
        };
        //以dataurl的形式读取图片文件
        reader.readAsDataURL(file);
    }
});
```

#### 2.压缩照片

```
//定义照片的最大高度
var MAX_HEIGHT = 480;
var render = function(src){
    var image = new Image();
	image.onload = function(){
		var cvs = document.getElementById("cvs");
		var w = image.width;
		var h = image.height;
		//计算压缩后的图片长和宽
		if(h>MAX_HEIGHT){
			w *= MAX_HEIGHT/h;
			h = MAX_HEIGHT;
		}
		var ctx = cvs.getContext("2d");
		cvs.width = w;
		cvs.height = h;
		//将图片绘制到Canvas上，从原点0,0绘制到w,h
		ctx.drawImage(image,0,0,w,h);
		
		//进入图片上传逻辑
		sendImg();
	};
	image.src = src;
};
```

#### 3.上传照片

```
var sendImg = function(){
    var cvs = document.getElementById("cvs");
    //调用Canvas的toDataURL接口，得到的是照片文件的base64编码string
	var data = cvs.toDataURL("image/jpeg");
	//base64 string过短显然就不是正常的图片数据了，过滤の。
	if(data.length<48){
		console.log("image data error.");
		return;
	}
	//图片的base64 string格式是data:/image/jpeg;base64,xxxxxxxxxxx
	//是以data:/image/jpeg;base64,开头的，我们在服务端写入图片数据的时候不需要这个头！
	//所以在这里只拿头后面的string
	//当然这一步可以在服务端做，但让闲着蛋疼的客户端帮着做一点吧~~~（稍微减轻一点服务器压力）
	data = data.split(",")[1];
	$.post("./api/uploadimg",{
		fileName:"xxx.jpeg",
		fileData:data
	},function(data){
		if(data.status==200){
		    // some code here.
			console.log("commit image success.");
		}else{
			console.log("commit image failed.");
		}
	},"json");
};
```

看完上面的代码，是不是觉得也没那么难？真的是这样吗？code旅途艰辛，显然没那么容易就让你好过。

测试后发现，在pc上以及大部分android和iphone4s+上是正常的，但是极小部分android和iphone4s以下的机型上得到的照片居然是不完整的！比如只有上半部分，下半部分是黑的，或者照片是旋转的！开始以为是服务端图片存储的时候出了问题，不过后面排除了服务端的问题，看来上面代码是有兼容性问题的。

具体排除问题的过程很复杂纠结，就不细说了。贴几个帖子：

1.[HTML5 Canvas drawImage ratio bug iOS](http://stackoverflow.com/questions/11929099/html5-canvas-drawimage-ratio-bug-ios)

2.[iOS HTML5 canvas drawImage vertical scaling bug, even for *small* images?](http://stackoverflow.com/questions/19432269/ios-html5-canvas-drawimage-vertical-scaling-bug-even-for-small-images)

3.[Drawing on canvas after megapix rendering is reversed](http://stackoverflow.com/questions/24998317/drawing-on-canvas-after-megapix-rendering-is-reversed)

主要是低版本的ios safari上面对于大尺寸的照片（超过设备的物理像素）处理的bug，导致的现象就是上半部分是照片下半部分是黑的，我们需要一个工具将一张大图切成若干个小于屏幕尺寸的小图，分别对小图进行处理然后再合并成一张图片。原理很简单，但实现起来就没那么简单了，还是已经有相关的开源工具来完成这个工作。

[Fixes iOS6 Safari's image file rendering issue for large size image (over mega-pixel), which causes unexpected subsampling when drawing it in canvas.](https://github.com/stomita/ios-imagefile-megapixel)

剩下一个图片旋转的问题，其实每张图片拍摄后`EXIF`里面都带有旋转`Orientation`字段来标注图片的旋转信息的，也就是说其实图片本身就是倒着的，但是图片展示的时候通过读取`Orientation`来修正图片展示，使图片能按照拍摄的角度展示，所以我们在写入图片数据的时候需要按照图片本身的`Orientation`来写入数据，这样我们就需要拿到图片本身的`EXIF`信息。

[JavaScript library for reading EXIF image metadata](https://github.com/jseidelin/exif-js)

ok，问题终于全部排除完毕啦。那么经过优化后的完整代码就是：

```
//绑定input change事件
$("#photo").unbind("change").on("change",function(){
    var file = this.files[0];
    if(file){
        //验证图片文件类型
        if(file.type && !/image/i.test(file.type)){
            return false;
        }
        var reader = new FileReader();
        reader.onload = function(e){
            //readAsDataURL后执行onload，进入图片压缩逻辑
            //e.target.result得到的就是图片文件的base64 string
            render(file,e.target.result);  
        };
        //以dataurl的形式读取图片文件
        reader.readAsDataURL(file);
    }
});

//定义照片的最大高度
var MAX_HEIGHT = 480;
var render = function(file,src){
	EXIF.getData(file,function(){
	    //获取照片本身的Orientation
		var orientation = EXIF.getTag(this, "Orientation");
		var image = new Image();
		image.onload = function(){
			var cvs = document.getElementById("cvs");
			var w = image.width;
			var h = image.height;
			//计算压缩后的图片长和宽
			if(h>MAX_HEIGHT){
				w *= MAX_HEIGHT/h;
				h = MAX_HEIGHT;
			}
			//使用MegaPixImage封装照片数据
			var mpImg = new MegaPixImage(file);
			//按照Orientation来写入图片数据，回调函数是上传图片到服务器
			mpImg.render(cvs, {maxWidth:w,maxHeight:h,orientation:orientation}, sendImg);
		};
		image.src = src;
	});
};

//上传图片到服务器
var sendImg = function(){
    var cvs = document.getElementById("cvs");
    //调用Canvas的toDataURL接口，得到的是照片文件的base64编码string
    var data = cvs.toDataURL("image/jpeg");
    //base64 string过短显然就不是正常的图片数据了，过滤の。
    if(data.length<48){
    	console.log("data error.");
    	return;
    }
    //图片的base64 string格式是data:/image/jpeg;base64,xxxxxxxxxxx
    //是以data:/image/jpeg;base64,开头的，我们在服务端写入图片数据的时候不需要这个头！
    //所以在这里只拿头后面的string
    //当然这一步可以在服务端做，但让闲着蛋疼的客户端帮着做一点吧~~~（稍微减轻一点服务器压力）
    data = data.split(",")[1];
    $.post("./api/uploadimg",{
    	fileName:"xxx.jpeg",
    	fileData:data
    },function(data){
    	if(data.status==200){
            // some code here.
            console.log("commit image success.");
        }else{
            console.log("commit image failed.");
        }
    },"json");
};
```

ok，结果一番折腾后，终于在所有设备上都能正常运行了:)



