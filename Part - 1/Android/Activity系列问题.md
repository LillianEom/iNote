# Activity系列问题

### 1.介绍下不同场景下Activity生命周期的变化过程启动Activity**：**

onCreate()--->onStart()--->onResume()，Activity进入运行状态。

**Activity退居后台**： 当前Activity转到新的Activity界面或按Home键回到主屏： onPause()--->onStop()，进入停滞状态。

**Activity返回前台**： onRestart()--->onStart()--->onResume()，再次回到运行状态。

**Activity退居后台，且系统内存不足**， 系统会杀死这个后台状态的Activity，若再次回到这个Activity,则会走onCreate()-->onStart()--->onResume()

**锁定屏与解锁屏幕** 只会调用onPause()，而不会调用onStop方法，开屏后则调用onResume()

### 2.内存不足时系统会杀掉后台的Activity，若需要进行一些临时状态的保存，在哪个方法进行？

Activity的 onSaveInstanceState() 和 onRestoreInstanceState()并不是生命周期方法，它们不同于 onCreate()、onPause()等生命周期方法，它们并不一定会被触发。当应用遇到意外情况（如：内存不足、用户直接按Home键）由系统销毁一个Activity，onSaveInstanceState() 会被调用。但是当用户主动去销毁一个Activity时，例如在应用中按返回键，onSaveInstanceState()就不会被调用。除非该activity是被用户主动销毁的，通常onSaveInstanceState()只适合用于保存一些临时性的状态，而onPause()适合用于数据的持久化保存。

### 3. onSaveInstanceState() 被执行的场景有哪些：

系统不知道你按下HOME后要运行多少其他的程序，自然也不知道activity A是否会被销毁，因此系统都会调用onSaveInstanceState()，让用户有机会保存某些非永久性的数据。以下几种情况的分析都遵循该原则当用户按下HOME键时长按HOME键，选择运行其他的程序时锁屏时从activity A中启动一个新的activity时屏幕方向切换时

### 4. 介绍Activity的几中启动模式，并简单说说自己的理解或者使用场景

说一下打开一个activity的过程[https://spring2613.github.io/2016/03/05/start-activity/](https://spring2613.github.io/2016/03/05/start-activity/)

* **standard**：这是默认模式，每次激活Activity时都会创建Activity实例，并放入任务栈中。
* **singleTop**： 如果在任务的栈顶正好存在该Activity的实例，就重用该实例( 会调用实例的 onNewIntent() )，否则就会创建新的实例并放入栈顶，即使栈中已经存在该Activity的实例，只要不在栈顶，都会创建新的实例。
* **singleTask**：如果在栈中已经有该Activity的实例，就重用该实例(会调用实例的 onNewIntent() )。重用时，会让该实例回到栈顶，因此在它上面的实例将会被移出栈。如果栈中不存在该实例，将会创建新的实例放入栈中。
* **singleInstance**：在一个新栈中创建该Activity的实例，并让多个应用共享该栈中的该Activity实例。一旦该模式的 Activity实例已经存在于某个栈中，任何应用再激活该Activity时都会重用该栈中的实例( 会调用实例的 onNewIntent() )。其效果相当于多个应用共享一个应用，不管谁激活该 Activity 都会进入同一个应用中。

备注：其使用方式为：在AndroidManifest.xml 文件中 Activity 元素的 android:launchMode 属性上设置 为 上面列出的某一种属性值即可。



自己测试了一下 singleTask在栈里没有实例的时候 是会直接放到栈顶 还是会另开一个栈

A-》B（singleTask）-》C 经测试 taskID并没有发生改变

A-》B（singleInstance）-》C 经测试，A与C均在task21中 只有B在task22中A-》B（singleInstance）-》C 然后C后退 后退到的是A A再后退，居然回了B，再后退是完全退出了
11、详细展开说一下所有LaunchMode的应用场景

假如A-》B-》C，想让C后退直接到A，使用什么样的intentflag？

假如A-》B-》C，C使用singleTask，C后退，后退到什么地方呢？

所以第一个是B是singleInstance可以达成

第二个 C后退 后退到的还是B singleTask并没有另开一个栈Intent intent= new Intent(context, YourActivity.class);intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_SINGLE_TOP);

LaunchMode 这一点我之前也是简简单单准备的有四种，每种什么意思，然后网易让我发现了，大家都知道这些，重点是要知道如何应用。美团这里问到的是我A打开了B，B打开了C，C的右上角有一个叉叉，那我怎样做到使我点击C的叉叉，就直接关闭了BC回到A，PS，不可以用startActivityForResult

https://developer.android.com/guide/components/tasks-and-back-stack.html

http://blog.sina.com.cn/s/blog_6b6f6a800102vwah.htmlhttp://souly.cn/%E6%8A%80%E6%9C%AF%E5%8D%9A%E6%96%87/2015/07/03/activity-LaunchMode-%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF/



### 19. 同一个应用程序的不同Activity可以运行在不同的进程中么？如果可以，举例说明

```java
public class ActivityStack {   
    ......   
    private final void startSpecificActivityLocked(ActivityRecord r,   
            boolean andResume, boolean checkConfig) {   
        // Is this activity's application already running?   
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,   
            r.info.applicationInfo.uid);   
        ......   
        if (app != null && app.thread != null) {   
            try {   
                realStartActivityLocked(r, app, andResume, checkConfig);   
                return;   
            } catch (RemoteException e) {   
                ......   
            }   
        }   
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,   
            "activity", r.intent.getComponent(), false);   
    }   
    ......
}   
```