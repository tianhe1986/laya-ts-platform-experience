# QQ小游戏
QQ底层应该是做了一些兼容，laya ide中直接发布成微信小游戏，是可以运行的，但是我接入的时候，有一些问题需要做些调整。如果后面的版本已经修复了相关的问题，就没必要特殊处理了。

# 资源缓存未生效
每次进入小游戏都需要重新加载。经调试后发现问题出在`laya.wxmini.js`的`MiniFileMgr.existDir`中。
在微信小游戏中，调用`mkdirSync`的时候，如果文件夹已经存在，会抛出错误，从而进入catch分支中，读取缓存文件信息。
但是在QQ小游戏中，文件夹已存在的情况下，没有抛出错误，从而未进入catch分支，也就未能读取缓存文件信息，造成资源重新从网络加载一遍。
我这边做了如下修改，将
```
MiniFileMgr.fs.mkdirSync(dirPath,true);
callBack && callBack.runWith([0,{data:'{}'}]);
return;
```
一段改为
```
MiniFileMgr.fs.mkdirSync(dirPath,true);
MiniFileMgr.readSync(MiniFileMgr.fileListName,'utf8',callBack);
return;
```
强制尝试读取资源列表信息，这样就可以了。

# IOS开放数据域获取用户信息
在IOS真机上调用开放域的`getUserInfo`时，会取不到openId，从而无法将openId和用户信息相对应。
经观察后发现，取到的用户信息中，avatar的格式是有规律的，其中有一段就是相应的openId。
一种作法是，通过取相应位置的子串得到openId。
另一种作法是，将openId排序，获取到的信息按avatar排序，之后一一对应。不过前提是没有传`selfOpenId`。

# banner广告
虽然可以设置宽高，但是会自行缩放，我这边的情况是以宽度为准，高度缩到约是宽度的1/4。
banner广告在onResize中重新设置left和top不会生效，所以我这边都是在初始化的时候直接设置。希望之后更新能修复此问题。
在IOS中，如果banner广告想要靠底，top应该设为（系统的screenHeight - banner高度），不能用windowHeight。