# 理解 ThreadLocal

## ThreadLocal 是什么

ThreadLocal 是一个关于创建线程局部变量的类。

通常情况下，我们创建的变量是可以被任何一个线程访问并修改的。而使用 ThreadLocal 创建的变量只能被当前线程访问，其他线程则无法访问和修改。

**应用场景：**

一般来说，当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用 ThreadLocal。比如对于 Handler 来说，它需要获取当前线程的 Looper，很显然 Looper 的作用域就是线程并且不同线程具有不同的 Looper，这个时候通过 ThreadLocal 就可以轻松实现 Looper 在线程中的存取，如果不采用 ThreadLocal，那么系统就必须提供一个全局的哈希表供 Handler 查找指定线程的 Looper，这样一来就必须提供一个类似于 LooperManager 的类了，但是系统并没有这么做而是选择了 ThreadLocal，这就是 ThreadLocal 的好处。

## Global && Local

上面的两个修饰看似矛盾，实则不然。

- Global 意思是在当前线程中，任何一个点都可以访问到ThreadLocal的值。
- Local 意思是该线程的ThreadLocal只能被该线程访问，一般情况下其他线程访问不到。

## 用法简介

### 创建，支持泛型

`ThreadLocal<String> mStringThreadLocal = new ThreadLocal<>();`

### set 方法

`mStringThreadLocal.set("test");`

### get 方法

`mStringThreadLocal.get();`

### 完整的使用示例

```java
private void testThreadLocal() {
    Thread t = new Thread() {
        ThreadLocal<String> mStringThreadLocal = new ThreadLocal<>();

        @Override
        public void run() {
            super.run();
            mStringThreadLocal.set("test");
            mStringThreadLocal.get();
        }
    };

    t.start();
}
```

### ThreadLocal 初始值

为 ThreadLocal 设置默认的 get 初始值，需要重写`initialValue`方法，下面是一段代码，我们将默认值修改成了线程的名字

```java
ThreadLocal<String> mThreadLocal = new ThreadLocal<String>() {
    @Override
    protected String initialValue() {
      return Thread.currentThread().getName();
    }
};
```



## Android中的应用

在Android中，Looper 类就是利用了 ThreadLocal 的特性，保证每个线程只存在一个 Looper对象。

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```



## 如何实现

为了更好的掌握 ThreadLocal，我认为了解其内部实现是很有必要的，这里我们以 set 方法从起始看一看 ThreadLocal 的实现原理。

下面是 ThreadLocal 的 set 方法，大致意思为

- 首先获取当前线程
- 利用当前线程作为句柄获取一个 ThreadLocalMap 的对象
- 如果上述 ThreadLocalMap 对象不为空，则设置值，否则创建这个 ThreadLocalMap 对象并设置值

源码如下

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

下面是一个利用 Thread 对象作为句柄获取 ThreadLocalMap 对象的代码

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

上面的代码获取的实际上是 Thread 对象的 threadLocals 变量，可参考下面代码

```java
class Thread implements Runnable {
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */

    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

而如果一开始设置，即 ThreadLocalMap 对象未创建，则新建 ThreadLocalMap 对象，并设置初始值。

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

总结：实际上 ThreadLocal 的值是放入了当前线程的一个 ThreadLocalMap 实例中，所以只能在本线程中访问，其他线程无法访问。

## 对象存放在哪里

在Java中，栈内存归属于单个线程，每个线程都会有一个栈内存，其存储的变量只能在其所属线程中可见，即栈内存可以理解成线程的私有内存。而堆内存中的对象对所有线程可见。堆内存中的对象可以被所有线程访问。

### 问：那么是不是说ThreadLocal的实例以及其值存放在栈上呢？

其实不是，因为 ThreadLocal 实例实际上也是被其创建的类持有（更顶端应该是被线程持有）。而 ThreadLocal 的值其实也是被线程实例持有。

**它们都是位于堆上，只是通过一些技巧将可见性修改成了线程可见。**

关于堆和栈的比较，请参考[Java中的堆和栈的区别](http://droidyue.com/blog/2014/12/07/differences-between-stack-and-heap-in-java/)

## 真的只能被一个线程访问么

既然上面提到了 ThreadLocal 只对当前线程可见，是不是说 ThreadLocal 的值只能被一个线程访问呢？

使用 InheritableThreadLocal 可以实现多个线程访问 ThreadLocal 的值。

如下，我们在主线程中创建一个 InheritableThreadLocal 的实例，然后在子线程中得到这个 InheritableThreadLocal 实例设置的值。

```java
private void testInheritableThreadLocal() {
    final ThreadLocal threadLocal = new InheritableThreadLocal();
    threadLocal.set("droidyue.com");
    Thread t = new Thread() {
        @Override
        public void run() {
            super.run();
            Log.i(LOGTAG, "testInheritableThreadLocal =" + threadLocal.get());
        }
    };

    t.start();
}
```

上面的代码输出的日志信息为

```
I/MainActivity( 5046): testInheritableThreadLocal =droidyue.com
```

使用 InheritableThreadLocal 可以将某个线程的 ThreadLocal 值在其子线程创建时传递过去。因为在线程创建过程中，有相关的处理逻辑。

```java
//Thread.java
 private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
        //code goes here
        if (parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
}
```

上面代码就是在线程创建的时候，复制父线程的 inheritableThreadLocals的数据。

## 会导致内存泄露么

有网上讨论说 ThreadLocal 会导致内存泄露，原因如下

- 首先 ThreadLocal 实例被线程的 ThreadLocalMap 实例持有，也可以看成被线程持有。
- 如果应用使用了线程池，那么之前的线程实例处理完之后出于复用的目的依然存活
- 所以，ThreadLocal 设定的值被持有，导致内存泄露。

上面的逻辑是清晰的，可是 ThreadLocal 并不会产生内存泄露，因为 ThreadLocalMap 在选择 key 的时候，并不是直接选择 ThreadLocal 实例，而是 ThreadLocal 实例的弱引用。

```java
static class ThreadLocalMap {

/**
* The entries in this hash map extend WeakReference, using
* its main ref field as the key (which is always a
* ThreadLocal object).  Note that null keys (i.e. entry.get()
* == null) mean that the key is no longer referenced, so the
* entry can be expunged from table.  Such entries are referred to
* as "stale entries" in the code that follows.
*/
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
}
```

所以实际上从 ThreadLocal 设计角度来说是不会导致内存泄露的。关于弱引用，了解更多，请访问[译文：理解Java中的弱引用](http://droidyue.com/blog/2014/10/12/understanding-weakreference-in-java/)

## 使用场景

- 实现单个线程单例以及单个线程上下文信息存储，比如交易 id 等
- 实现线程安全，非线程安全的对象使用 ThreadLocal 之后就会变得线程安全，因为每个线程都会有一个对应的实例
- 承载一些线程相关的数据，避免在方法中来回传递参数

**注意：Android 的 ThreadLocal 与 Java 实现略有不同，但是原理是一致的**。