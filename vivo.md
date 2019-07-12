# vivo小游戏
花了一两天时间定位问题及调整，改动了好几处。修改后的文件见[laya.vvmini.js](./replace/laya.vvmini.js)

# 修复未加载问题
将两句`indexOf('VVGame')<0`改为`indexOf('vivo')<0)`，打印`window.navigator.userAgent`可以看到原因。

# 修复storage报错
`MiniLocalStorage`中，对于写入和读取的调用方法跟vivo文档不一致，我这里懒得一个个修复了，直接注释掉这两句，发现原有的LocalStorage可以正常使用，就将就用了:)
```
LocalStorage._baseClass=MiniLocalStorage$4;
MiniLocalStorage$4.__init__();
```

# 网络文件缓存问题
主要有如下几处更改。
1. `writeFile`调用传参有误，将键名`text`传成了`data`，导致未写入缓存索引文件。
2. `readFileSync`调用结果处理有误，未能正确处理返回值类型，导致文件读取失败。
3. `readFile`处理有误，根据文档，临时文件不能直接读取，需要先拷贝到cache文件夹下。

# 其他注意事项
由于我们项目的需要，我这边修改了资源加载的方法`MiniLoader$4.load`，将文件都下载到了本地。如果不需要，可以还原这部分修改。