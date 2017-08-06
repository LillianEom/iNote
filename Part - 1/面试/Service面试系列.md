# Service系列问题

### 注册Service需要注意什么

**Service还是运行在主线程当中的，所以如果需要执行一些复杂的逻辑操作，最好在服务的内部手动创建子线程进行处理，否则会出现UI线程被阻塞的问题**

### Service与Activity怎么实现通信

* 方法一

  添加一个继承Binder的内部类，并添加相应的逻辑方法重写Service的onBind方法，返回我们刚刚定义的那个内部类实例Activity中创建一个ServiceConnection的匿名内部类，并且重写里面的onServiceConnected方法和onServiceDisconnected方法，这两个方法分别会在Activity与service成功绑定以及解除绑定的时候调用，在onServiceConnected方法中，我们可以得到一个刚才那个service的binder对象，通过对这个binder对象进行向下转型，得到我们那个自定义的Binder实例，有了这个实例，做可以调用这个实例里面的具体方法进行需要的操作了


* 方法二：

  通过BroadCast(广播)的形式 当我们的进度发生变化的时候我们发送一条广播，然后在Activity的注册广播接收器，接收到广播之后更新视图

### IntentService与Service的区别

IntentService是Service的子类，是一个异步的，会自动停止的服务，很好解决了传统的Service中处理完耗时操作忘记停止并销毁Service的问题会创建独立的worker线程来处理所有的Intent请求；

会创建独立的worker线程来处理onHandleIntent()方法实现的代码，无需处理多线程问题；所有请求处理完成后，IntentService会自动停止，无需调用stopSelf()方法停止Service；

为Service的onBind()提供默认实现，返回null；

为Service的onStartCommand提供默认实现，将请求Intent添加到队列中；IntentService不会阻塞UI线程，而普通Serveice会导致ANR异常Intentservice若未执行完成上一次的任务，将不会新开一个线程，是等待之前的任务完成后，再执行新的任务，等任务完成后再次调用stopSelf()

### service的生命周期

http://blog.csdn.net/agods/article/details/7468431

http://blog.csdn.net/android_tutor/article/details/5789203

使用context.startService() 启动Service

其生命周期为context.startService() ->onCreate()- >onStart()->Service running-->(如果调用context.stopService() )->onDestroy() ->Service shut down

如果Service还没有运行，则android先调用onCreate()然后调用onStart()；

如果Service已经运行，则只调用onStart()，所以一个Service的onStart方法可能会重复调用多次。 调用stopService的时候直接onDestroy，

如果是调用者自己直接退出而没有调用stopService的话，Service会一直在后台运行。

该Service的调用者再启动起来后可以通过stopService关闭Service。

所以调用startService的生命周期为：onCreate --> onStart(可多次调用) --> onDestroy对于bindService()启动Service会经历：

context.bindService()->onCreate()->onBind()->Service running-->onUnbind() -> onDestroy() ->Service stop onBind将返回给客户端一个IBind接口实例，IBind允许客户端回调服务的方法，比如得到Service运行的状态或其他操作。

这个时候把调用者（Context，例如Activity）会和Service绑定在一起，Context退出了，

Srevice就会调用onUnbind->onDestroy相应退出。 所以调用bindService的生命周期为：onCreate --> onBind(只一次，不可多次绑定) --> onUnbind --> onDestory。

一但销毁activity它就结束，如果按home把它放到后台，那他就不退出。
PS：

在Service每一次的开启关闭过程中，只有onStart可被多次调用(通过多次startService调用)，其他onCreate，onBind，onUnbind，onDestory在一个生命周期中只能被调用一次。

### service和线程的关系

http://www.cnblogs.com/newcj/archive/2011/05/30/2061370.html

Thread：Thread 是程序执行的最小单元，它是分配CPU的基本单位。可以用 Thread 来执行一些异步的操作。

Service：Service 是android的一种机制，当它运行的时候如果是Local Service，那么对应的 Service 是运行在主进程的 main 线程上的。如：onCreate，onStart 这些函数在被系统调用的时候都是在主进程的 main 线程上运行的。如果是Remote Service，那么对应的 Service 则是运行在独立进程的 main 线程上。因此请不要把 Service 理解成线程，它跟线程半毛钱的关系都没有！

既然这样，那么我们为什么要用 Service 呢？其实这跟 android 的系统机制有关，我们先拿 Thread 来说。Thread 的运行是独立于 Activity 的，也就是说当一个 Activity 被 finish 之后，如果你没有主动停止 Thread 或者 Thread 里的 run 方法没有执行完毕的话，Thread 也会一直执行。因此这里会出现一个问题：当 Activity 被 finish 之后，你不再持有该 Thread 的引用。另一方面，你没有办法在不同的 Activity 中对同一 Thread 进行控制。

Service组件主要有两个目的：后台运行和跨进程访问。service可以在android系统后台独立运行，线程是不可以。


举个例子：如果你的 Thread 需要不停地隔一段时间就要连接服务器做某种同步的话，该 Thread 需要在 Activity 没有start的时候也在运行。这个时候当你 start 一个 Activity 就没有办法在该 Activity 里面控制之前创建的 Thread。因此你便需要创建并启动一个 Service ，在 Service 里面创建、运行并控制该 Thread，这样便解决了该问题（因为任何 Activity 都可以控制同一 Service，而系统也只会创建一个对应 Service 的实例）。



因此你可以把 Service 想象成一种消息服务，而你可以在任何有 Context 的地方调用 Context.startService、Context.stopService、Context.bindService，Context.unbindService，来控制它，你也可以在 Service 里注册 BroadcastReceiver，在其他地方通过发送 broadcast 来控制它，当然这些都是 Thread 做不到的。



怎么让一个service不死掉，一直运行



Service和推送通知，通知有没有可能出现有推送但是通知栏收不到通知？service被kill掉会如何？如何保证service不被kill掉