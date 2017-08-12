# App运行是否在前台

在一些场景中，经常会需要判断App是否在后台运行，比如是否显示解锁界面，收到新消息是否显示Notification等。

其实实现手势密码的核心还是在于手势密码的**触发机制**,这一点就涉及到应用在**前台与后台之间切换状态**的监控了.



## 不当方法

for loop判断当前runningProcess或者runningTasks



## 合适的几种

Android在SDK 14的时候提供了一个Callback。ActivityLifecycleCallbacks，你可以通过这个Callback拿到App所有Activity的生命周期回调。

这个Callback写在Application里的，你可以在Application初始化的时候来注册。我们可以写个单例类来cache这些status。这里我叫它AppStatusTracker。在Application的onCreate()里让AppStatusTracker注册ActivityLifecycleCallbacks。



通过这样的判断，我们来利用ActivityLifecycleCallbacks回调的onActivityResumed()和onActivityPaused()方法来计数，如果只有一个resumeCount，那么当前App在前台，如果木有resumeCount，它就在后台。

**但是很快你会发现，这里有个延时，会导致判断不准确。**

我们假设有两个Activities，一个A，一个B，从A跳转到B，生命周期怎么走的？ A.onPause() -> B.onResume() 对应到ActivityLifecycleCallbacks里是onActivityPaused(A) -> onActivityResumed(B)，刚才我们说的计数resumeCount，在onActivityPaused()里--，在onActivityResumed()里++， 根据这样的判断会有个短暂的间隔，也就是在A的onPause()到B的onResume()之间，App是运行在后台的，这样逻辑肯定就不对了。



从WelcomeActivity跳转到GestureActivity。这里只说onStart, onResume这些回调

A.onPause() -> B.onStart() -> B.onResume() -> A.onStop()

通过这些回调我们可以将这个计数放在onStart()和onStop()中去，这样就不会存在那个短暂间隔。resumeCount==1，那么就是前台，resumeCount==0，那就是后台。这样判断很很简单了吧。



我们拆分下需求，先说锁屏，解锁。

这个是有BroadCastReciever来接收的，注册下就可以了，每次收到锁屏ACTION_SCREEN_OFF的action时，将AppStatusTracker里的isScreenOff设置为true。

当onActivityResumed()被调用时再将isScreenOff设为false。

再说切换到后台10分钟后显示手势解锁。这个只需要在onActivityStop()时更新下lastBackgroudTimestamp就可以了



## 实现步骤

稍微逛了一下简书和CSDN,发现在监控应用前后台切换状态方面也有几种实现方式,本文选择一种比较简单的方式进行说明.

1.首先是需要继承Application类实现自己的的自定义Application.

2.在自定义的Application的onCreate()方法中使用**registerActivityLifecycleCallbacks()**方法,该方法引用的匿名类中的方法可以实现对应用的所有Activity进行状态统计,从而达到监控应用前后台切换状态的效果.

3.加入一些判断条件,例如在我们的应用中就加入了判断应用是冷启动还是热启动的条件,从而达到实现某些特定的需求.



### 另一种

然后，核心逻辑要放在Activity进入后台和从后台恢复的生命周期回调中。网上有帖子说放onStop和onStart中，这是不可行的，在有Activity切换动画的情况下，前一个Activity的onStop是在另一个Activity的onStart之后才调用的，这个逻辑使得这两个方法不适合作为逻辑的入口。我们放到onResume和onPause中。现在，你的BaseActivity应该长这样。



最基本的逻辑是，（L1）给BaseActivity定义一个静态成员变量`sLastBackTime` ，在onPause中保存时间戳，在onResume中取这个时间戳和当前时间对比，如果超过了手势密码延迟的阈值（TH），就打开手势密码页面。