## 微信Mars——xlog使用全解析

[微信终端跨平台组件 mars 系列（一） - 高性能日志模块xlog](http://mp.weixin.qq.com/s/jBO29pk_yl4CQhuCxPQEMQ)



## 本地编译 so 库

下载好 [NDK ](https://developer.android.com/ndk/downloads/index.html?hl=zh-cn) 解压并配置好环境变量 path，然后确认两件事：

* python 版本为2.7x,
* 使用的 ndk 版本为 ndk-r11c 以上

clone 下 [Mars](https://github.com/Tencent/mars) 的源码，进入 libraries目录，直接执行下面的[Python](http://lib.csdn.net/base/python)脚本：

```
python build_android.py
```

```
Enter menu:
1. build mars shared libs.
2. build mars static libs.
3. build xlog shared lib with crypt.
4. exit.
```

编译 3， 输出结果全部在 `mars_xlog_sdk` 目录中。

![](http://om4rextnc.bkt.clouddn.com/17-7-21/99994971.jpg)

把 `mars_android_sdk/src` 目录下的 Java 文件以及 libs/ 复制到你的项目中。

## 使用

### 权限

xlog可以加密每一行输出的文件并写入文件，所以需要下面的权限：

```
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

### 加载so

推荐在程序启动时加载 xlog：

```
System.loadLibrary("stlport_shared");
System.loadLibrary("marsxlog");
```

### 初始化

在程序启动加载 xlog 后紧接着初始化 xlog:

```
final String SDCARD = Environment.getExternalStorageDirectory().getAbsolutePath();
final String logPath = SDCARD + "/marssample/log";

// this is necessary, or may cash for SIGBUS
final String cachePath = this.getFilesDir() + "/xlog"

//init xlog
if (BuildConfig.DEBUG) {
    Xlog.appenderOpen(Xlog.LEVEL_DEBUG, Xlog.AppednerModeAsync, cachePath, logPath, "MarsSample", PUB_KEY);
    Xlog.setConsoleLogOpen(true);

} else {
    Xlog.appenderOpen(Xlog.LEVEL_INFO, Xlog.AppednerModeAsync, cachePath, logPath, "MarsSample", PUB_KEY);
    Xlog.setConsoleLogOpen(false);
}

Log.setLogImp(new Xlog());
```

### 停止Log记录

在 Application 或者 Activity 的销毁方法中，进行 xlog 的关闭操作，从而生成日志文件：

```
Log.appenderClose();
```

<div class = "tip">

- 如果你的程序使用了多进程，不要把多个进程的日志输出到同一个文件中，保证每个进程独享一个日志文件。
- 保存 log 的目录请使用单独的目录，不要存放任何其他文件防止被 xlog 自动清理功能误删。
- debug 版本下建议把控制台日志打开，日志级别设为 Verbose 或者 Debug, release 版本建议把控制台日志关闭，日志级别使用 Info.
- cachePath 这个参数必传，而且要data下的私有文件目录，例如 /data/data/packagename/files/xlog， mmap文件会放在这个目录，如果传空串，可能会发生 SIGBUS 的crash。

</div>

## 解析 Log

Log 生成完毕后，会在指定的路径下生成相应的日志文件：

```shell
shell@R7:/sdcard/marssample/log $ ll
-rw-rw---- root     sdcard_r   153600 2016-12-30 17:06 MarsSample.mmap2
-rw-rw---- root     sdcard_r    29633 2016-12-30 17:06 MarsSample_20161230.xlog
```

其中MarsSample.mmap2 是缓存文件，不用关心，我们需要的是.xlog文件，我们把这个文件pull出来，使用Mars提供的[python](http://lib.csdn.net/base/python)脚本进行解密。

解压脚本在 Mars 源码 `log/crypt/decode_mars_log_file.py` 下的这个文件，执行：

```shell
mars_xlog_sdk python decode_mars_log_file.py ~/Downloads/log/MarsSample_20161230.xlog
```

即可生成对应的log文件。

具体问题可见 Wiki

[Mars 常见问题](https://github.com/Tencent/mars/wiki/Mars-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)

[Mars Android 接入指南](https://github.com/Tencent/mars/wiki/Mars-Android-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97)

[xlog 常见问题issues](https://github.com/Tencent/mars/issues?utf8=%E2%9C%93&q=xlog)











