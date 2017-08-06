# Handler

### 1.android消息处理机制， Handler、Looper、MessageQueue，handler+loop分析；

### **2.handler.postAtTime不是延时post么 那handler怎么延时处理Message**

sendMessageDelayed()：发送一个Message对象到消息队列中，在UI线程取到消息后，延迟执行

android 中发送消息不管是Message中的几种重载的obtain()方式，还是Handler中的几种重载的sendMessage最终都是通过Handler.sendMessage来发送的,而Handler中的几种sendMessage()重载方法最终都会调用到sendMessageAtTime()方法来完成消息的入队操作。

### 3.looper是线程级别还是进程级别

A、Looper 线程就是循环工作的线程； 

B、一个Thread只能有一个Looper对象，它是一个ThreadLocal;

C、一个线程可以有多个Handler，但是只能有一个Looper

### 4.loop线程与普通线程区别

Looper线程与标准线程的区别在于：Looper线程的run方法执行后并不会立即退出，而是进入一个Loop消息循环中等待消息的到来，然后根据消息类型分别作出不同的处理。这样，后台只需运行单一线程，当需要定时或者不定时执行不同任务时，分别发送不同的消息给Looper线程去处理，这样就避免了频繁 创建/销毁线程所带来的开销。

### 说一下HandlerThread

http://gold.xitu.io/entry/5791db935bbb500063c18cc2

这个类的作用是创建一个包含looper的线程。

那么我们在什么时候需要用到它呢?加入在应用程序当中为了实现同时完成多个任务，所以我们会在应用程序当中创建多个线程。为了让多个线程之间能够方便的通信，我们会使用Handler实现线程间的通信。这个时候我们手动实现的多线程+Handler的简化版就是我们HandlerThrea所要做的事了。


我们发现其内部调用了Looper.prepate()方法和Loop.loop()方法，熟悉android异步消息机制的童鞋应当知道，在android体系中一个线程其实是对应着一个Looper对象、一个MessageQueue对象，以及N个Handler对象，具体可参考：android源码解析之（二）–>异步消息机制


所以通过run方法，我们可以知道在我们创建的HandlerThread线程中我们创建了该线程的Looper与MessageQueue；


这里需要注意的是其在调用Looper.loop()方法之前调用了一个空的实现方法：onLooperPrepared(),我们可以实现自己的onLooperPrepared（）方法，做一些Looper的初始化操作；
run方法里面当mLooper创建完成后有个notifyAll()，getLooper()中有个wait()，这是为什么呢？因为的mLooper在一个线程中执行，而我们的handler是在UI线程初始化的，也就是说，我们必须等到mLooper创建完成，才能正确的返回getLooper();wait(),notify()就是为了解决这两个线程的同步问题

该Handler的构造方法中传入了HandlerThread的Looper对象，所以Handler对象就相当于含有了HandlerThread线程中Looper对象的引用。


然后我们调用handler的sendMessage方法发送消息，在Handler的handleMessge方法中就可以接收到消息了。
最后需要注意的是在我们不需要这个looper线程的时候需要手动停止掉；
总结：HandlerThread本质上是一个Thread对象，只不过其内部帮我们创建了该线程的Looper和MessageQueue；
通过HandlerThread我们不但可以实现UI线程与子线程的通信同样也可以实现子线程与子线程之间的通信；

HandlerThread在不需要使用的时候需要手动的回收掉；
http://blog.csdn.net/lmj623565791/article/details/47079737

其实我们就是初始化和启动了一个线程；然后我们看run()方法，可以看到该方法中调用了Looper.prepare()，Loop.loop();prepare()呢，中创建了一个Looper对象，并且把该对象放到了该线程范围内的变量中（sThreadLocal），在Looper对象的构造过程中，初始化了一个MessageQueue，作为该Looper对象成员变量。loop()就开启了，不断的循环从MessageQueue中取消息处理了，当没有消息的时候会阻塞，有消息的到来的时候会唤醒。
那么Handler的构造呢，其实就是在Handler中持有一个指向该Looper.mQueue对象，当handler调用sendMessage方法时，其实就是往该mQueue中去插入一个message，然后Looper.loop()就会取出执行。

**如果你够细心你会发现，run方法里面当mLooper创建完成后有个notifyAll()，getLooper()中有个wait()，这是为什么呢？因为的mLooper在一个线程中执行，而我们的handler是在UI线程初始化的，也就是说，我们必须等到mLooper创建完成，才能正确的返回getLooper();wait(),notify()就是为了解决这两个线程的同步问题。**

### 你的APP里，是每个Activity都有一个Handler呢还是所有Activity共享一个Handler

1. 整个主UI ，只创建一个handler,即全局的handler.然后多个activity共享这一个handler,发送消息。

   优点： 只用一个消息循环，比较能提高性能。

   缺点： 发送消息时，传递数据不方便。需要将activity的 各变量值打包 传递给 这个全局的handler.



2. 每一个Activity创建一个handler, 当前Activity就用它自己的handler变量来发送消息。

   优点：发送消息时，传递数据很方便，因为就利用当前activity类里的变量值。

   缺点： 创建了多个消息队列，容易忘掉释放，影响性能。

