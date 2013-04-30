---
layout: default
title: HTML5 Ajax Canvas缩放图片后后变成文件并上传的方法
---
#{{ page.title }}
最近做的应用里面有上传图片的功能，由于图片直接丢进七牛云存储，所以不想通过服务器端程序来写缩放功能，查了一下找到一篇文章，[点我](http://hacks.mozilla.org/2011/01/how-to-develop-a-html5-image-uploader/) ，这篇总体步骤来说是OK的，但是里面有好多坑。

本来准备把上传的独立一篇写出来的，后来发现太短，加到后面吧。

PS：想直接看完整可以用版代码的请直接拖到最下。

首先创建Img并画到Canvas上这一步，他的代码连起来是这样的：

{% highlight javascript %}
var img = document.createElement("img");
img.src = window.URL.createObjectURL(file);
var MAX_WIDTH = 800;
var MAX_HEIGHT = 600;
var width = img.width;
var height = img.height;
if (width > height) {
    if (width > MAX_WIDTH) {
        height *= MAX_WIDTH / width;
        width = MAX_WIDTH;
    }
} else {
    if (height > MAX_HEIGHT) {
        width *= MAX_HEIGHT / height;
        height = MAX_HEIGHT;
    }
}
canvas.width = width;
canvas.height = height;
var ctx = canvas.getContext("2d");
ctx.drawImage(img, 0, 0, width, height);
{% endhighlight %}

但是这样写Canvas里面什么都不会出现，我昨天跟这段代码战斗了小半天，才发现他应该改成：

{% highlight javascript %}
var img = document.createElement("img");
img.onload = function(){
	var MAX_WIDTH = 800;
	var MAX_HEIGHT = 600;
	var width = img.width;
	var height = img.height;
	if (width > height) {
  		if (width > MAX_WIDTH) {
    		height *= MAX_WIDTH / width;
    		width = MAX_WIDTH;
    	}
	} else {
  		if (height > MAX_HEIGHT) {
    		width *= MAX_HEIGHT / height;
    		height = MAX_HEIGHT;
    	}
	}
	canvas.width = width;
	canvas.height = height;
	var ctx = canvas.getContext("2d");
	ctx.drawImage(img, 0, 0, width, height);
}
img.src = window.URL.createObjectURL(file);
{% endhighlight %}

这是因为Javascript是异步调用的，正序执行的时候img还没画出来，ctx.drawImage已经调用了，导致什么效果都没有。

上面那段代码的最后一句也是个坑，在Chrome，Safari下都报错，其实应该写成：

{% highlight javascript %}
var myURL = window.URL || window.webkitURL
img.src = myURL.createObjectURL(file);
{% endhighlight %}

然后要将Canvas转换成文件来上传，这篇写了这两个方法：

{% highlight javascript %}
var dataurl = canvas.toDataURL("image/png");
var file = canvas.mozGetAsFile("foo.png");
{% endhighlight %}

然而实际上mozGetAsFile只有Firefox支持，所以为了兼容Chrome跟Safari我去查了一下dataurl怎么用。这个原文就找不到了，也被坑了一次，搜索到的前几条好多BlobBuilder的，这玩意不知道什么时候就deprecated了，最新版的Chrome跟Safari都没这类，最终找到了一个ArrayBuffer直接转换blob的代码：

{% highlight javascript %}
function dataURItoBlob(dataURI) {
    var byteString = atob(dataURI.split(',')[1]);
    var mimeString = dataURI.split(',')[0].split(':')[1].split(';')[0]
    var ab = new ArrayBuffer(byteString.length);
    var ia = new Uint8Array(ab);
    for (var i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i);
    }
    var dataView = new DataView(ab)
    var blob = new Blob([dataView], { type: mimeString })
    return blob
}
{% endhighlight %}

这段代码在Chrome里面用的毫无问题，但是Safari生成的blob只有十几字节，这明显不科学。[MDN Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob?redirectlocale=en-US&redirectslug=DOM%2FBlob) 上查到可以直接把ArrayBuffer传到Blob构造函数里面，于是就变成了

{% highlight javascript %}
var blob = new Blob([ab], { type: mimeString })
{% endhighlight %}

现在圆满了，就剩一个小问题，Chrome下会提示把ArrayBuffer传入Blob构造函数的行为deprecated了。

接下来就是文件上传了，用到了jQuery。

{% highlight javascript %}
var formData = new FormData()
formData.append("file", imgFile)
$.ajax('http://example.com/upload',{
    type: 'POST',
    contentType: 'multipart/form-data;',
    xhr: function() {  // 自定义xhr
        myXhr = $.ajaxSettings.xhr();
        var eventSource = myXhr.upload || myXhr;
        eventSource.addEventListener('progress',progressHandler); // 处理上传进度，大文件的时候很有必要
        return myXhr;
    },
    //Ajax events
    success: successHandler,
    error: function(jqXHR, textStatus, errorThrown){
        console.log(errorThrown)
    },
    data: formData,
    cache: false,
    contentType: false,
    processData: false
})
function progressHandler(event){
    var percent = Math.round( progressEvent.loaded * 100 / progressEvent.total);
    //你的进度条处理方法
}
{% endhighlight %}

这个没啥坑，有个小问题，按之前的办法生成的Blob拿过来上传的时候在Firefox里面无法显示Progress，应该是Firefox只有上传文件才有prgress事件。

以下是完整可用代码：

{% highlight javascript %}
fileinput.onchange = getImgFile //file input的change事件绑定
var imgFile
function getImgFile(event){
    var img = document.createElement("img")
    img.onload = function(){
        var canvas = document.getElementById("coverFileCanvas")
        var ctx = canvas.getContext("2d")
        ctx.drawImage(img, 0, 0, WIDTH, HEIGHT)
        if(canvas.mozGetAsFile){
            imgFile = canvas.mozGetAsFile("img.png")
        }else{
            var dataURL = canvas.toDataURL("image/png")
            var filenameParts = coverfile.name.split(".")
            var extension = filenameParts[filenameParts.length-1]
            imgFile = dataURItoBlob(dataURL)
        }
    }
    var myURL = window.URL || window.webkitURL
    img.src = myURL.createObjectURL(event.target.file[0])
}
function dataURItoBlob(dataURI) {
    var byteString = atob(dataURI.split(',')[1]);
    var mimeString = dataURI.split(',')[0].split(':')[1].split(';')[0]
    var ab = new ArrayBuffer(byteString.length);
    var ia = new Uint8Array(ab);
    for (var i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i);
    }
    var blob = new Blob([ab], { type: mimeString })
    return blob
}

function fileUpload(){
    var formData = new FormData()
    formData.append("file", imgFile)
    $.ajax('http://example.com/upload',{
        type: 'POST',
        contentType: 'multipart/form-data;',
        xhr: function() {  // 自定义xhr
            myXhr = $.ajaxSettings.xhr();
            var eventSource = myXhr.upload || myXhr;
            eventSource.addEventListener('progress',progressHandler); // 处理上传进度，大文件的时候很有必要
            return myXhr;
        },
        //Ajax events
        success: successHandler,
        error: function(jqXHR, textStatus, errorThrown){
            console.log(errorThrown)
        },
        data: formData,
        cache: false,
        contentType: false,
        processData: false
    })
}
function progressHandler(event){
    var percent = Math.round( progressEvent.loaded * 100 / progressEvent.total);
    //你的进度条处理方法
}
{% endhighlight %}

<p>{{ page.date | date_to_string }}</p>