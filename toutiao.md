# 头条小游戏
同样可以直接发布成微信小游戏，头条底层应该也做了相应的适配，但是我接入的时候，坑还比如多，希望之后版本更新，一点点完善起来。

# 使用panel造成的IOS花屏问题
如果用了panel，而所在的view位于最上层的话，其下层的3d渲染在头条IOS版本中会出现花屏。
我的处理时用box + rect mask替换。

# PlaneMesh对象宽高属性更改造成的问题
在安卓上会造成相应的MeshSprite3D物体无法展示，IOS则会直接退出游戏。
以更改width为例，经代码阅读及调试之后，关键处理步骤如下：
1.  调用this.releaseResource();
    * 调用this.disposeResource(), 调用父类PrimitiveMesh.disposeResource()， 调用this._vertexBuffer和this._indexBuffer的destroy方法
    * destroy两个buffer时，调用父类Resource.destroy， 调用this.releaseResource，再调用Buffer.disposeResource
2.	调用this.activeResource();
    * 调用Resource.activeResource, 调用this.recreateResource, new 重新生成this._vertexBuffer和this._indexBuffer
    * 生成两个buffer时，
        * 调用this._bind, 调用this.activeResource，调用Buffer.recreateResource，调用Buffer._gl.createBuffer即WebGL.mainContext.createBuffer
        * 调用Buffer._gl.bufferData

个人的猜测是，在releaseResource过程中，destroy掉buffer时，Buffer.disposeResource调用`WebGL.mainContext.deleteBuffer`。
在activeResource过程中，再次生成buffer时,Buffer.recreateResource调用`WebGL.mainContext.createBuffer`，得到的还是刚刚删除的同一对象。

这样在`_bind`时，就不会再次调用bindBuffer的操作，导致之后的bufferData写入出现问题。

修改方法是修改`laya.webgl.js`中Buffer的`disposeResource`方法。
之前是
```
__proto.disposeResource=function(){
    if (this._glBuffer){
        WebGL.mainContext.deleteBuffer(this._glBuffer);
        this._glBuffer=null;
    }
    this.memorySize=0;
}
```
增加一句解绑的操作，改为
```
__proto.disposeResource=function(){
    if (this._glBuffer){
        if (Buffer._bindActive[this._bufferType]===this._glBuffer) {
            Buffer._gl.bindBuffer(this._bufferType,Buffer._bindActive[this._bufferType]=null);
        }
        WebGL.mainContext.deleteBuffer(this._glBuffer);
        this._glBuffer=null;
    }
    this.memorySize=0;
}
```

# 头像展示的问题
头条头像的二级域名比较多,而laya中加载图片默认使用的是下载后再使用本地url的方式,这样的话配置download file白名单会有问题.
我这边改了下`laya.wxmini.js`文件,对于头条头像域名'pstatp.com'的URL,不使用下载,直接创建图片对象。

具体做法如下：
在`MiniImage`的`_loadImage`方法中,找到
```
MiniFileMgr.downOtherFiles(XXXX)
```
这一句,加上个判断改成
```
if (url.indexOf("pstatp.com")==-1 || ! Laya.Browser.window.tt) {
    MiniFileMgr.downOtherFiles(XXXX)
} else {
    MiniImage.onCreateImage(url,thisLoader,true);
}
```

后来，又发现头像使用了其他的域名，索性将黑名单机制改成了白名单机制，只下载特定域名URL的图片资源，其他的都直接创建图片对象，即
```
if ( ! Laya.Browser.window.tt || url.indexOf("XXX") != -1 || url.indexOf("XXX") != -1) {
    MiniFileMgr.downOtherFiles(XXXX);
} else {
    MiniImage.onCreateImage(url,thisLoader,true);
}
```

# Texture问题
在某天头条版本更新后，我发现我这边的某个小游戏打不开了，报错信息为`activeResource undefined`。
最终调试定位，发现是在`Texture`的`__getset(0,__proto,'source`中，`this.bitmap`不是Resource对象，console.log输出是一个canvas。
由于是p0级错误，因此直接采取了简单粗暴让游戏可以运行的方法。
将
```
this.bitmap.activeResource();
```
改成
```
this.bitmap.activeResource && this.bitmap.activeResource();
```
~~但具体的canvas出现的地方，还需要进一步研究重现，之后有结论了会进行更新。~~

经过研究发现，问题出现的地方是在主域中设定子域canvas大小的时，使用了`new Laya.Texture(Laya.Browser.window.sharedCanvas)`.
然后在`Texture`的`setTo`方法中，`bitmap instanceof window.HTMLElement`判断失败，直接赋值，导致this.bitmap不是Resource对象。

打印调试发现，`window.sharedCanvas`，也就是头条的`tt.getOpenDataContext().canvas`,是经过处理的。它虽然是`HTMLCanvasElement`，但却不是window定义的`HTMLCanvasElement`，所以判断失败。

因此，除了上面的简单粗暴的方法外，还有另一种简单粗暴的方法。
在`Texture`的`setTo`方法中，将
```
if (/*__JS__ */bitmap instanceof window.HTMLElement)
```
改为
```
if (/*__JS__ */bitmap instanceof window.HTMLElement || (Laya.Browser.window.tt && bitmap === Laya.Browser.window.tt.getOpenDataContext().canvas))
```

# 需要接入录屏功能
头条的录屏是必须要接入的，不要忘记。

# 分享奖励onShow调整
~~分享奖励，通常是通过监听`onShow`事件中处理的。而在头条里，调起选择平台界面时，还未能触发`onShow`事件，此时点击取消，则监听事件还在，而代码里没有办法获取到这次点击取消的行为，也就无法立马取消对分享的监听。
我这边的做法有些麻烦，是在其他按钮的事件里取消`onShow`发放奖励的行为。~~

头条类库更新后，可以直接从`tt.shareAppMessage`中的success和fail回调得知分享成功与否，不再需要用`tt.onShow`事件处理，如：
```
tt.shareAppMessage({
    title: "XXX",
    imageUrl: "SSS",
    success() {
        console.log("分享成功");
    },
    fail(e) {
        console.log("分享失败");
    }
})
```

# banner广告调整
头条广告的宽高是固定比例16:9，需要重新计算并置于合适的位置。

# 激励视频
跟微信里没有太大的差别，但是开发者工具中调用创建激励视频组件时会出错，需要判断兼容一下。

# 授权调整
头条是可以直接请求用户授权获取用户信息的，因此没必要创建原生的授权按钮。