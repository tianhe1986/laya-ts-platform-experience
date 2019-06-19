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

# 需要接入录屏功能

# 分享奖励onShow调整

# banner广告调整
