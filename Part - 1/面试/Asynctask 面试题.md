# Asynctask 面试题

### 一、Asynctask源码分析

**这里有个线程池几个关键参数5,128，使用的阻塞队列，补充讲了asynctask缺陷**

http://blog.csdn.net/lmj623565791/article/details/38614699

简单说来asynctask就是**thread+handler+线程池**

背后原来有一个线程池，且最大支持128的线程并发，加上长度为10的阻塞队列，可能会觉得就是在快速调用138个以内的AsyncTask子类的execute方法不会出现问题，而大于138则会抛出异常。

其实不是这样的，我们再仔细看一下代码，回顾一下sDefaultExecutor，真正在execute()中调用的为sDefaultExecutor.execute：

如果此时有10个任务同时调用execute（s synchronized）方法，第一个任务入队，然后在mActive = mTasks.poll()) != null被取出，并且赋值给mActivte，然后交给线程池去执行。然后第二个任务入队，但是此时mActive并不为null，并不会执行scheduleNext();所以如果第一个任务比较慢，10个任务都会进入队列等待；真正执行下一个任务的时机是，线程池执行完成第一个任务以后，调用Runnable中的finally代码块中的scheduleNext，所以虽然内部有一个线程池，其实调用的过程还是线性的。一个接着一个的执行，相当于单线程。

### 二、AsyncTask运行的原理是什么？有什么缺陷？


以前对于缺陷的答案可能是：AsyncTask在并发执行多个任务时发生异常。其实还是存在的，在3.0以前的系统中还是会以支持多线程并发的方式执行，支持并发数也是我们上面所计算的128，阻塞队列可以存放10个；也就是同时执行138个任务是没有问题的；而超过138会马上出现Java.util.concurrent.RejectedExecutionException；

而在在3.0以上包括3.0的系统中会为单线程执行（即我们上面代码的分析）；

如果现在大家去面试，被问到AsyncTask的缺陷，可以分为两个部分说，在3.0以前，最大支持128个线程的并发，10个任务的等待。在3.0以后，无论有多少任务，都会在其内部单线程执行；

还分析了AsyncTask的缺点，就是它所维护的线程池大小为128，同一时刻只能有5个工作线程和一个缓存线程，如果耗时操作工作量巨大就会导致线程池大小不够用，这就是它的缺点，另外我还介绍了它的解决方式，就是由一个控制线程来处理AsyncTask的调用，判断线程池是否已经满了，如果满的话就停止处理。

### 三、AsyncTask 内部实现机理 与Thread+Handler有什么不同

本质上来说，AsyncTask就是用于解决异步处理任务的类，而它的内部实现是Thread+Handler的组合，题主可能会问了，那肥肥鱼大大说的线程池为啥也被引入AsyncTask了呢？主要原因在于，我们在异步处理任务的时候可能需要进行多线程异步处理，那么每次都要手动去创建线程，以及手动管理这些线程都会非常麻烦，而Java在解决并发编程问题时提出的线程池类则能很好地解决这类问题，于是AsyncTask里就有了这个类了。
很多网友可能发现Android平台很多应用使用的都是AsyncTask，而并非Thread和Handler去更新UI，这里给大家说下他们到底有什 么区别，我们平时应该使用哪种解决方案。从Android 1.5开始系统将AsyncTask引入到android.os包中，过去在很早1.1和1.0 SDK时其实官方将其命名为UserTask，其内部是JDK 1.5开始新增的concurrent库，做过J2EE的网友可能明白并发库效率和强大性，比Java原始的Thread更灵活和强大，但对于轻量级的使 用更为占用系统资源。Thread是Java早期为实现多线程而设计的，比较简单不支持concurrent中很多特性在同步和线程池类中需要自己去实现 很多的东西，对于分布式应用来说更需要自己写调度代码，而为了Android UI的刷新Google引入了Handler和Looper机制，它们均基于消息实现，有时可能消息队列阻塞或其他原因无法准确的使用。

推荐大家使用AsyncTask代替Thread+Handler的方式，不仅调用上更为简单，经过实测更可靠一些，Google在Browser中大量 使用了异步任务作为处理耗时的I/O操作，比如下载文件、读写数据库等等，它们在本质上都离不开消息，但是AsyncTask相比Thread加 Handler更为可靠，更易于维护，但AsyncTask缺点也是有的比如一旦线程开启即dobackground方法执行后无法给线程发送消息，仅能 通过预先设置好的标记来控制逻辑，当然可以通过线程的挂起等待标志位的改变来通讯，对于某些应用Thread和Handler以及Looper可能更灵 活。

### 四、为什么java已经有线程池的实现了，还要继续使用AsyncTask，AsyncTask相对于java自带的线程池的好处

AsyncTask 最大的优势是可以把结果返回给主线程 你用thread写很难写对

缺点就是设计来做比较短的工作的 最多几秒钟 底下的文档有提到

AsyncTask是android提供的轻量级的异步类,可以直接继承AsyncTask,在类中实现异步操作,并提供接口反馈当前异步执行的程度(可以通过接口实现UI进度更新),最后反馈执行的结果给UI主线程.


Android为了降低这个开发难度，提供了AsyncTask。AsyncTask就是一个封装过的后台任务类，顾名思义就是异步任务。

AsyncTask直接继承于Object类，位置为android.os.AsyncTask。要使用AsyncTask工作我们要提供三个泛型参数，并重载几个方法(至少重载一个)。

### 五、AsynTask造成的内存泄露的问题怎么解决

从而延伸到 jvm虚拟机四种引用


跨线程通信

夸进程通信 主要是Android的AIDL
在Activity中定义AsyncTask导致内存泄漏

由于AsyncTask是Activity的内部类，所以会持有外部的一个引用，如果Activity已经退出，但是AsyncTask还没有执行完毕，那么Activity就无法释放导致内存泄漏。对于这个问题我们可以把AsyncTask定义为静态内部类并且采用弱引用。

**比如非静态内部类AsynTask会隐式地持有外部类的引用，如果其生命周期大于外部activity的生命周期，就会出现内存泄漏注意要复写AsynTask的onCancel方法，把里面的socket，file等，该关掉的要及时关掉在 Activity 的onDestory()方法中调用Asyntask.cancal方法Asyntask内部使用弱引用的方式来持有Activity**

### 六、AsynTask为什么要设计为只能够一次任务？

**最核心的还是线程安全问题，多个子线程同时运行，会产生状态不一致的问题。所以要务必保证只能够执行一次**



### 若Activity已经销毁，此时AsynTask执行完并且返回结果，会报异常吗?

**当一个App旋转时，整个Activity会被销毁和重建。当Activity重启时，AsyncTask中对该Activity的引用是无效的，因此onPostExecute()就不会起作用，若AsynTask正在执行，折会报 view not attached to window manager 异常同样也是生命周期的问题，在 Activity 的onDestory()方法中调用Asyntask.cancal方法，让二者的生命周期同步**

### Activity销毁但Task如果没有销毁掉，当Activity重启时这个AsyncTask该如何解决？

**还是屏幕旋转这个例子，在重建Activity的时候，会回掉Activity.onRetainNonConfigurationInstance()重新传递一个新的对象给AsyncTask，完成引用的更新**



**AsyncTask在4.x以后有什么改变？怎样改回并发执行多个？如果一个AsyncTask结束取得结果之前Activity就因为内存原因被Destroy掉了，那会有什么情况发生？会内存泄露吗？会空指针吗？需要在Activity彻底死掉之前把AsyncTask cancel掉吗？如果没有cancel掉，然后Activity重启了，那这个Asynctask又当如何呢？**

在3.0版本以前AsyncTask的线程只有1个线程池，核心线程数为5，最大线程数为128，任务队列容量为10。

也就是说当线程池中的线程数量没到5个，那么有新的任务会直接启动一个核心线程来执行任务，如果线程池中的线程数量达到了5个，然后任务会被插入到任务队列中等待执行，要是任务队列也满了，就会判断线程池中的数量是否已经达到最大线程数128，如果没有达到就会立刻启动一个非核心线程来执行任务。如果线程数量已经达到线程池规定的最大值，那么就会拒绝执行该任务。也就是说该线程池最多能同时接纳138个任务，其中有128个任务可以同时执行。而且该版本只有一个execute(Params... params) 方法，说明不能自定义线程池来执行任务。



4.4版本以后的线程池数量改为了动态的，以双核心为例，先获取CPU的核心数为2，线程池的核心线程为3，最大线程数为5,而阻塞队列的容量变为了128。为什么会有这样的改动？可能谷歌公司觉得开启的线程数过多会影响效率吧。而阻塞队列从容量为10变为了128是一个很有意思的事情。在4.4以前的版本，如果已经达到了线程池的核心线程数5，切阻塞队列也达到了10，再有任务加入，就会启动新的非核心线程，也就是说只要同时又16个任务进入就会开启非核心线程。而现在需要132（3+128+1）个任务加入才会开启非核心线程。也就是说要开启新的线程的成本更大了。



不能及时取消任务

http://www.jianshu.com/p/e60c3bb03d61

以4.4版本双核手机为例，如果用户在A界面发起5个任务，由于使用SerialExecutor来执行任务，那么任务将一个一个顺序执行，由于第一个任务执行时间过长，其他任务只能在队列中等待，导致阻塞，所以可以考虑把执行时间较短的任务优先加入。如果在第一个任务执行过程中，用户跳转到了B界面，而A界面发起的任务已经没有必要执行，所以我们要在Activity的生命周期结束的时候取消掉任务。如果任务没取消掉，B界面又发起新的任务，就会导致B界面的所有请求阻塞。如果有需要我们可以直接使用executeOnExecutor方法，然后直接使用THREAD_POOL_EXECUTOR线程池来执行，这样可以3个线程同时执行，并且在doInBackground方法中判断任务是否被取消，这样可以提高效率。



http://blog.csdn.net/singwhatiwanna/article/details/17596225

并发执行：

我已经给出了在3.0以上系统中让AsyncTask并行执行的方法，现在，让我们来试一试，代码还是之前采用的测试代码，我们要稍作修改，调用AsyncTask的executeOnExecutor方法而不是execute




1、生命周期


很多开发者会认为一个在Activity中创建的AsyncTask会随着Activity的销毁而销毁。然而事实并非如此。AsyncTask会一直执行, 直到doInBackground()方法执行完毕。然后，如果 cancel(boolean)被调用, 那么onCancelled(Result result) 方法会被执行；否则，执行onPostExecute(Result result) 方法。如果我们的Activity销毁之前，没有取消 AsyncTask，这有可能让我们的AsyncTask崩溃(crash)。因为它想要处理的view已经不存在了。所以，我们总是必须确保在销毁活动之前取消任务。总之，我们使用AsyncTask需要确保AsyncTask正确地取消。


另外，即使我们正确地调用了cancle() 也未必能真正地取消任务。因为如果在doInBackgroud里有一个不可中断的操作，比如BitmapFactory.decodeStream()，那么这个操作会继续下去。


2、内存泄漏


如果AsyncTask被声明为Activity的非静态的内部类，那么AsyncTask会保留一个对创建了AsyncTask的Activity的引用。如果Activity已经被销毁，AsyncTask的后台线程还在执行，它将继续在内存里保留这个引用，导致Activity无法被回收，引起内存泄露。


3、结果丢失

屏幕旋转或Activity在后台被系统杀掉等情况会导致Activity的重新创建，之前运行的AsyncTask会持有一个之前Activity的引用，这个引用已经无效，这时调用onPostExecute()再去更新界面将不再生效。

