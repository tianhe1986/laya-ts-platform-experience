# 通用
这里会讲一些所有平台都有可能碰到的问题。

# 本地缓存文件失效
在某些平台（至今为止，我在头条和vivo碰到过）下，可能由于APP内部（也有可能是手机系统）的清理，造成本地缓存的文件丢失，但是保存缓存对应关系的`layaairfiles.txt`文件没有丢失。

这样，以图片文件为例，当需要加载远程图片文件且使用缓存时，就会尝试根据对应关系，去加载保存在本地的文件，而此时文件已经不在了，因此加载失败。

那么，问题来了，本地文件加载失败之后，做什么呢？现在默认的行为是，再次尝试加载。而关键就在于，再次尝试加载，还是按之前同样的方式进行，查对应关系，加载本地文件，再失败，再重试。

我这边做的改动是，当本地文件加载失败时，清除本地缓存关系表中的相关信息，再尝试加载，这样，对于缓存丢失的情况，就会重新尝试从网络加载了。

以微信适配文件`laya.wxmini.js`为例，具体的做法是，将`MiniImage.onCreateImage`方法中的这一段
```
        var onerror=function (){
			clear();
			thisLoader.event(/*laya.events.Event.ERROR*/"error","Load image failed");
		}
```

改为

```
        var onerror=function (){
			clear();
			MiniFileMgr.onSaveFile(sourceUrl,"",false,"", Handler.create(null,function(dummyCode){
				thisLoader.event(/*laya.events.Event.ERROR*/"error","Load image failed");
			}));
		}
```

# 缓存其他文件
默认情况下，仅缓存图片和声音资源，不缓存其他格式的文件。而实际过程中，我们可能需要缓存其他类型的文件，那此时怎么办呢？

还是以`laya.wxmini.js`为例，我们要做的是修改`MiniLoader.load`方法，具体的地方是这一段
```
                var tempUrl=url;
				url=URL.formatURL(url);
				if (url.indexOf("http://")!=-1 || url.indexOf("https://")!=-1){
					MiniAdpter.EnvConfig.load.call(thisLoader,tempUrl,type,cache,group,ignoreCache);
					}else {
					MiniFileMgr.readFile(url,encoding,new Handler(MiniLoader,MiniLoader.onReadNativeCallBack,[encoding,url,type,cache,group,ignoreCache,thisLoader]),url);
				}
```

可以看到，如果是从网络加载，就会去调用Laya默认的加载方法，使用XMLHttpRequest直接获取文件内容，不会下载到本地了。

将其修改为以下几步即可，参考`MiniFileMgr.downOtherFiles`方法：
1. 调用`MiniFileMgr.wxdown`下载文件到本地。
2. 调用`MiniFileMgr.copyFile`处理缓存。
3. 调用`MiniFileMgr.readFile`读取本地文件获取内容，触发加载完成回调。

由于我们的项目还做了不少其他的改动，导致现在的文件与原版相差甚远，就暂时先不上传了，我会尽快整理出一份只修改了缓存资源部分的文件放上来。