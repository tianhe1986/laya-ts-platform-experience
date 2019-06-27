# 头条小游戏
同样可以直接发布成微信小游戏，头条底层应该也做了相应的适配，但是我接入的时候，坑还比如多，希望之后版本更新，一点点完善起来。

# 使用panel造成的IOS花屏问题
如果用了panel，而所在的view位于最上层的话，其下层的3d渲染在头条IOS版本中会出现花屏。
我的处理时用box + rect mask替换。

# PlaneMesh属性更改造成的问题
在安卓上会造成相应的MeshSprite3D物体无法展示，IOS则会直接退出游戏。
经代码调试之后，个人的猜测是，Buffer对象dispose时调用`WebGL.mainContext.deleteBuffer`，再次recreate创建时调用`createBuffer`，得到的还是刚刚删除的同一对象。
这样在接下来调用`_bind`时，就不会再次调用bindBuffer的操作，导致之后的bufferData写入出现问题。
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

# 奇怪的Texture问题
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
但具体的canvas出现的地方，还需要进一步研究重现，之后有结论了会进行更新。

# 需要接入录屏功能
头条的录屏是必须要接入的，不要忘记。

# 分享奖励onShow调整
分享奖励，通常是通过监听`onShow`事件中处理的。而在头条里，调起选择平台界面时，还未能触发`onShow`事件，此时点击取消，则监听事件还在，而代码里没有办法获取到这次点击取消的行为，也就无法立马取消对分享的监听。
我这边的做法有些麻烦，是在其他按钮的事件里取消`onShow`发放奖励的行为。

# banner广告调整
头条广告的宽高是固定比例，需要重新计算并置于合适的位置。

# 激励视频
跟微信里没有太大的差别，但是开发者工具中调用创建激励视频组件时会出错，需要判断兼容一下。

# 授权调整
头条是可以直接请求用户授权获取用户信息的，因此没必要创建原生的授权按钮。