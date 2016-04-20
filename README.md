# wechatBug
微信JSSDK分享的一些坑
字数1258 阅读341 评论0 喜欢1
最近做微信方面的H5应用比较多，很多情况下需要定制页面的分享内容，例如标题、描述、分享的图片等，这时候就需要用到微信的JSSDK。
在开发的过程中遇到了一些不大不小的坑，今天在下面总结一下，以备将来之需。

坑一：朋友圈链接不刷新
最近做了一个需要动态改变分享图片的项目，A和B处于关注状态，当A把项目分享到朋友圈后，B从朋友圈中打开链接，并再次执行页面程序，当再次分享到朋友圈时，发现分享出去的图片仍是A分享出去的图片。

最开始是以为代码的问题，但是经过调试，排除了代码bug。
然后考虑可能是分享机制问题，遂除了用jssdk方式分享外，还加入了jsBridge方式，但是问题仍没有解决。

接着想到也许是服务器缓存造成的，但发现程序执行后的结果每次都是新的，所以排除了这种情况。

后来考虑是APP缓存的问题，一度因此打算改变项目策略，但发现类似的一个项目却没有出现这个问题，而且发现每次程序执行完后分享给朋友时图片是新的，只有从朋友圈里打开的链接不能让程序结果更新。

最后想到可能是页面没有刷新，所以给分享链接加了随机查询参数，以强制刷新。没料到，还真是！

所以如果你的项目中需要动态地改变分享出去的内容，最好在分享URL后面加入随机查询参数，这样在朋友圈里的链接地址都会不一样，不同的人打开同一个链接会强制刷新。

坑二：分享png图片黑底
还是上面那个例子，当程序生成png图片后预览时是没有问题的，但分享出去后只能看到一张黑色的图。

看到这个现象，马上想到可能是png图片不被微信支持。
但是转而就否定了这个观点，微信的浏览器再差也不可能不支持png图片吧！
于是想到可能是微信自动把png图片转了其他格式，至于什么格式，不知道。如果真是这样，那就无解了。

接着想也许是图片合成的原因，遂测试了几张不同背景和前景的图片，发现：如果背景是透明的，前景字体是红色的，则合成后的图片分享后能看到黑底红字；如果背景是白色，前景是红色，则分享后能看到白底红字。

所以能得出结论：如果微信分享出去的图片是透明背景png图片，则会看到黑底；如果是不透明的则不会看到黑底。
这样我只要生成不透明的背景的图片就行了，测试了几下，问题解决。

坑三：canvas跨域
其实这不能算是微信的坑，而是canvas的坑。因为在这个项目中出现了，一并提一下。
还是接着上面的坑说，为生成不透明的png图片，我选择预先给canvas写入图像。
首先我们定义一个函数：

/**
* @param _src 要载入的图像地址
* @param _canvasContext canvasContext对象
* @param _scale 要写入的尺寸数组[width,height]
*/
function loadImg2Canvas(_src,_canvasContext,_scale){
    var img = new Image();  
    img.src = _src; 
    if(img.complete){
        _canvasContext.drawImage(img, 0, 0,_scale[0],_scale[1]);
    }else{
        img.onload = function(){
            _canvasContext.drawImage(img, 0, 0,_scale[0],_scale[1]);
        };
        img.onerror = function(){
            window.alert('load failed');
        };
    }
}
然后我们给canvas对象载入图像地址：

var canvas  = document.getElementById("thecanvas");
var canvas_bg = "http://www.example.com/canvas_bg.jpg";
loadImg2Canvas(canvas_bg,ctx,[canvas.width,canvas.height]);
但是这个过程中出现了跨域错误：


canvas跨域错误

从网上查找，很多文章会建议你给图片所在的域的响应头中附加上 Access-Control-Allow-Origin: * 字段，并且给要加载的图像加上img.crossOrigin = "",或img.crossOrigin = "anonymous"，于是我们的函数就变成了：

/**
* @param _src 要载入的图像地址
* @param _canvasContext canvasContext对象
* @param _scale 要写入的尺寸数组[width,height]
*/
function loadImg2Canvas(_src,_canvasContext,_scale){
    var img = new Image();  
    img.src = _src; 
    img.crossOrigin = '';
    if(img.complete){
        _canvasContext.drawImage(img, 0, 0,_scale[0],_scale[1]);
    }else{
        img.onload = function(){
            _canvasContext.drawImage(img, 0, 0,_scale[0],_scale[1]);
        };
        img.onerror = function(){
            window.alert('load failed');
        };
    }
}
但是问题来了，很多情况下我们要加载的图片是从资源库或者CDN来的，这些图片服务器可能无法加上Access-Control-Allow-Origin: * 字段，所以这种情况下就无能为力了。

最终我只能把图像跟代码放到同一个域上，这样就不存在跨域的问题了。

期待有其他可能的解决办法！
