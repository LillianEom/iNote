# APP启动优化

## 启动类型

### 冷启动

当启动应用时，后台没有该应用的进程，这时系统会重新创建一个新的进程分配给该应用，这个启动方式就是冷启动。

特点：冷启动因为系统会重新创建一个新的进程分配给它，所以会先创建和初始化**application**类，再创建和初始化**MainActivity**类（包括一系列的测量、布局、绘制），最后显示在界面上。

### 热启动

当启动应用时，后台已有该应用的进程（例：按**back**键、**home**键，应用虽然会退出，但是该应用的进程是依然会保留在后台，可进入任务列表查看），所以在已有进程的情况下，这种启动会从已有的进程中来启动应用，那么应用程序可以避免重复对象初始化，UI的布局和渲染。

特点：热启动因为会从已有的进程中来启动，所以热启动就不会走**application**这步了，而是直接走**MainActivity**（包括一系列的测量、布局、绘制），所以热启动的过程只需要创建和初始化一个**MainActivity**就行了，而不必创建和初始化**application**，因为一个应用从新进程的创建到进程的销毁，**application**只会初始化一次。

### 温启动

用户退出您的应用，但随后重新启动。该过程可能已继续运行，但应用程序必须通过调用onCreate（）从头开始重新创建活动。系统从内存中驱逐您的应用程序，然后用户重新启动它。进程和Activity需要重新启动，但任务可以从保存的实例状态包传递到onCreate（）中。

## 为什么会出现黑白屏

首先我们要知道当打开一个Activity的时候发生了什么，在一个Activity打开时，如果该Activity所属的Application还没有启动，那么系统会为这个Activity创建一个进程（每创建一个进程都会调用一次Application，所以Application的onCreate()方法可能会被调用多次），在进程的创建和初始化中，势必会消耗一些时间，在这个时间里，WindowManager会先加载APP里的主题样式里的窗口背景（windowBackground）作为预览元素，然后才去真正的加载布局，如果这个时间过长，而默认的背景又是黑色或者白色，这样会给用户造成一种错觉，这个APP很卡，很不流畅，自然也影响了用户体验。

## 优化方案

### 直接干掉

既然有这个Activity启动界面，那能不能直接不要这个呢，当然是可以：
定义一个style：

```xml
<style name="AppTheme.Launcher">
    <!--关闭启动窗口-->
    <item name="android:windowDisablePreview">true</item>
</style>
```

只需要再启动页面引用：

```xml
<activity
    android:name=".MainActivity"
    android:label="@string/app_name"
    android:theme="@style/AppTheme.Launcher">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
```

最后在MainActivity恢复正常主题：

```java
public class MainActivity extends BaseActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setTheme(R.style.AppTheme);
        setContentView(R.layout.activity_main);
    }
}
```

这样启动APP，就没有白屏，但会出现点击桌面图标而半天没有反应的现象，显然不好，很多APP把这个闪屏当做一个广告、品牌宣传的页面。

### 设置透明主题

```xml
<style name="SplashTheme" parent="AppTheme">
    <item name="android:windowFullscreen">true</item>
    <item name="android:windowIsTranslucent">true</item>
</style>
```



### Theme

Material Design规范[launch-screens](https://material.io/guidelines/patterns/launch-screens.html)

1. 把启动图设置为窗体背景，避免刚刚启动App的时候出现，黑/白屏
2. 设置为背景显示的时候，后台负责加载资源，同时去下载广告图，广告图下载成功或者超时的时候显示`SplashActivity`的真实样子
3. 随后进入`MainAcitivity` 

屏幕提供短暂的品牌曝光，来看看如何实现的，定义一个style：

```xml
<style name="AppTheme.Launcher">
    <item name="android:windowBackground">@drawable/branded_launch_screens</item>
</style>
```

drawable/branded_launch_screens

```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:opacity="opaque">
    <!--黑色背景颜色-->
    <item android:drawable="@android:color/black" />
    <!-- 产品logo-->
    <item>
        <bitmap
            android:gravity="center"
            android:src="@mipmap/empty_image01" />
    </item>
    <!-- 右上角的图标元素 -->
    <item>
        <bitmap
            android:gravity="top|right"
            android:src="@mipmap/github" />
    </item>
    <!--最下面的文字-->
    <item android:bottom="50dp">
        <bitmap
            android:gravity="bottom"
            android:src="@mipmap/ic_launcher" />
    </item>
</layer-list>
```

其中android:opacity=”opaque”参数是为了防止在启动的时候出现背景的闪烁。关于layer-list介绍，见博客：用layer-list实现图片旋转叠加、错位叠加、阴影、按钮指示灯[http://www.cnblogs.com/tianzhijiexian/p/3889770.html](http://www.cnblogs.com/tianzhijiexian/p/3889770.html) ，也同样只需要再启动页面引用和最后在MainActivity恢复正常主题。

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        setTheme(R.style.AppTheme.Launcher);
        super.onCreate(savedInstanceState);
        SystemClock.sleep(2000);
        setContentView(R.layout.activity_main);
    }
}
```



官方文档Launch-Time Performance
[https://developer.android.google.cn/topic/performance/launch-time.html](https://developer.android.google.cn/topic/performance/launch-time.html)

[Launch-time]([https://www.youtube.com/watch?v=Vw1G1s73DsY&index=74&list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE](https://link.juejin.im/?target=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3DVw1G1s73DsY%26index%3D74%26list%3DPLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE))

[https://material.google.com/patterns/launch-screens.html](https://material.io/guidelines/patterns/launch-screens.htmll)

Android性能优化典范 - 第6季
[http://hukai.me/android-performance-patterns-season-6/](http://hukai.me/android-performance-patterns-season-6/)

[一触即发——App启动优化最佳实践](http://blog.csdn.net/eclipsexys/article/details/53044990)

[Android性能优化（一）之启动加速35%](https://juejin.im/post/5874bff0128fe1006b443fa0)





