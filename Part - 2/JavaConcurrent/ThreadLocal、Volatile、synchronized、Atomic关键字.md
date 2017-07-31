## ThreadLocal、Volatile、synchronized、Atomic关键字



## 1、Atomic

### 作用

对于原子操作类，Java 的 concurrent 并发包中主要为我们提供了这么几个常用的：AtomicInteger、AtomicLong、AtomicBoolean、AtomicReference<T>。 
对于原子操作类，最大的特点是在多线程并发操作同一个资源的情况下，使用 Lock-Free 算法来替代锁，这样开销小、速度快，对于原子操作类是采用原子操作指令实现的，从而可以保证操作的原子性。什么是原子性？比如一个操作 `i++`；实际上这是三个原子操作，先把 `i` 的值读取、然后修改`(+1)`、最后写入给 `i`。所以使用 Atomic 原子类操作数，比如：i++；那么它会在这步操作都完成情况下才允许其它线程再对它进行操作，而这个实现则是通过 Lock-Free+原子操作指令来确定的 

AtomicInteger类中：

```java
public final int incrementAndGet() {
    for (;;) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next))
          return next;
    }
}
```

而关于 Lock-Free 算法，则是一种新的策略替代锁来保证资源在并发时的完整性的，Lock-Free 的实现有三步：

> 1、循环（for(;;)、while） 
> 2、CAS（CompareAndSet） 
> 3、回退（return、break）

### 用法

比如在多个线程操作一个 count 变量的情况下，则可以把 count 定义为 AtomicInteger，如下：

```java
public class Counter {
    private AtomicInteger count = new AtomicInteger();

    public int getCount() {
        return count.get();
    }

    public void increment() {
        count.incrementAndGet();
    }
}
```

在每个线程中通过 increment() 来对 count 进行计数增加的操作，或者其它一些操作。这样每个线程访问到的将是安全、完整的 count。

### **内部实现**

采用 Lock-Free 算法替代锁+原子操作指令实现并发情况下资源的安全、完整、一致性

## 2、Volatile

### 作用

Volatile 可以看做是一个轻量级的 synchronized，它可以**在多线程并发的情况下保证变量的“可见性”**，什么是可见性？就是在**一个线程的工作内存中修改了该变量的值，该变量的值立即能回显到主内存中，从而保证所有的线程看到这个变量的值是一致的**。所以在**处理同步**问题上它大显作用，而且它的开销比 synchronized 小、使用成本更低。 
举个栗子：在写单例模式中，除了用静态内部类外，还有一种写法也非常受欢迎，就是 Volatile+DCL：

```java
public class Singleton {
    private static volatile Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

这样单例不管在哪个线程中创建的，所有线程都是共享这个单例的。

虽说这个 Volatile 关键字可以解决多线程环境下的同步问题，不过这也是相对的，因为它**不具有操作的原子性**，也就是它**不适合在对该变量的写操作依赖于变量本身自己**。举个最简单的栗子：在进行计数操作时 count++，实际是 count=count+1;，count 最终的值依赖于它本身的值。所以使用 volatile 修饰的变量在进行这么一系列的操作的时候，就有并发的问题 
举个栗子：因为它不具有操作的原子性，有可能 1 号线程在即将进行写操作时 count 值为 4；而 2 号线程就恰好获取了写操作之前的值 4，所以 1 号线程在完成它的写操作后 count 值就为 5 了，而在 2 号线程中 count 的值还为 4，即使 2 号线程已经完成了写操作 count 还是为 5，而我们期望的是 count 最终为 6，所以这样就有并发的问题。而如果 count 换成这样：count=num+1；假设 num 是同步的，那么这样 count 就没有并发的问题的，只要最终的值不依赖自己本身。

### 用法

因为 volatile 不具有操作的原子性，所以如果用 volatile 修饰的变量在进行依赖于它自身的操作时，就有并发问题，如：count，像下面这样写在并发环境中是达不到任何效果的：

```java
public class Counter {
    private volatile int count;

    public int getCount(){
        return count;
    }
    public void increment(){
        count++;
    }
}
```

而要想 count 能在并发环境中保持数据的一致性，则可以在 increment() 中加 **synchronized** 同步锁修饰，改进后的为：

```java
public class Counter {
    private volatile int count;

    public int getCount(){
        return count;
    }
    public synchronized void increment(){
        count++;
    }
}
```

### **内部实现**

使用 Violatile 修饰的变量在汇编阶段，会多出一条**lock 前缀指令**，lock 前缀指令实际上相当于一个内存屏障（也成内存栅栏），内存屏障会提供3 个功能：

1. 将当前处理器缓存行的数据写回到系统内存
2. 这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。
3. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；

通常处理器和内存之间都有几级缓存来提高处理速度，处理器先将内存中的数据读取到内部缓存后再进行操作，但是对于缓存写会内存的时机则无法得知，因此在一个处理器里修改的变量值，不一定能及时写会缓存，这种变量修改对其他处理器变得“不可见”了。但是，使用Volatile修饰的变量，在写操作的时候，会强制将这个变量所在缓存行的数据写回到内存中，但即使写回到内存，其他处理器也有可能使用内部的缓存数据，从而导致变量不一致，所以，在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期，如果过期，就会将该缓存行设置成无效状态，下次要使用就会重新从内存中读取。

## **3、synchronized**

### **作用**

synchronized 叫做**同步锁**，是 Lock 的一个简化版本，由于是简化版本，那么性能肯定是不如 Lock 的，不过它操作起来方便，**只需要在一个方法或把需要同步的代码块包装在它内部**，那么这段代码就是**同步**的了，**所有线程对这块区域的代码访问必须先持有锁才能进入**，否则则拦截在外面等待正在持有锁的线程处理完毕再获取锁进入，正因为它基于这种阻塞的策略，所以它的性能不太好，但是由于操作上的优势，只需要简单的声明一下即可，而且被它声明的代码块也是**具有操作的原子性**。

### **用法**

```java
public synchronized void increment(){
  count++;
}

public void increment(){
  synchronized (Counte.class){
    count++;
  }
}
```

### **内部实现**

重入锁 ReentrantLock+一个 Condition，所以说是 Lock 的简化版本，因为一个 Lock 往往可以对应多个 Condition

## **4、ThreadLocal**

### **作用**

关于 ThreadLocal，这个类的出现并不是用来解决在多线程并发环境下资源的共享问题的，它和其它三个关键字不一样，其它三个关键字都是从线程外来保证变量的一致性，这样使得多个线程访问的变量具有一致性，可以更好的体现出资源的共享。

而 ThreadLocal 的设计，并不是解决资源共享的问题，而是用来**提供线程内的局部变量**，这样每个线程都自己管理自己的局部变量，别的线程操作的数据不会对我产生影响，互不影响，所以不存在解决资源共享这么一说，如果是解决资源共享，那么其它线程操作的结果必然我需要获取到，而 ThreadLocal 则是自己管理自己的，相当于封装在 Thread 内部了，供线程自己管理。

### **用法**

一般使用 ThreadLocal，官方建议我们定义为 private static ，至于为什么要定义成静态的，这和内存泄露有关，后面再讲。 

它有三个暴露的方法，set、get、remove。

```java
public class ThreadLocalDemo {
    private static ThreadLocal<String> threadLocal = new ThreadLocal<String>(){
        @Override
        protected String initialValue() {
            return "hello";
        }
    };
    static class MyRunnable implements Runnable{
        private int num;
        public MyRunnable(int num){
            this.num = num;
        }
        @Override
        public void run() {
            threadLocal.set(String.valueOf(num));
            System.out.println("threadLocalValue:"+threadLocal.get());
        }
    }

    public static void main(String[] args){
        new Thread(new MyRunnable(1));
        new Thread(new MyRunnable(2));
        new Thread(new MyRunnable(3));
    }
}
```

运行结果如下，这些 ThreadLocal 变量属于线程内部管理的，互不影响：

> threadLocalValue:2 
> threadLocalValue:3 
> threadLocalValue:4

对于 get 方法，在 ThreadLocal 没有 set 值得情况下，默认返回 null，所有如果要有一个初始值我们可以重写 initialValue()方法，在没有 set 值得情况下调用 get 则返回初始值。

**值得注意的一点**：ThreadLocal 在线程使用完毕后，我们应该手动调用 remove 方法，移除它内部的值，这样可以防止内存泄露，当然还有设为static。

### **内部实现**

ThreadLocal 内部有一个静态类 ThreadLocalMap，使用到 ThreadLocal 的线程会与 ThreadLocalMap 绑定，维护着这个 Map 对象，而这个 ThreadLocalMap 的作用是映射当前 ThreadLocal 对应的值，它 key 为当前 ThreadLocal 的弱引用：WeakReference

#### **内存泄露问题**

对于 ThreadLocal，一直涉及到内存的泄露问题，即当该线程不需要再操作某个 ThreadLocal 内的值时，应该手动的 remove 掉，为什么呢？我们来看看 ThreadLocal 与 Thread 的联系图： 

此图来自网络： 
![这里写图片描述](http://img.blog.csdn.net/20160121000731607)

其中虚线表示弱引用，从该图可以看出，一个 Thread 维持着一个 ThreadLocalMap 对象，而该 Map 对象的 key 又由提供该 value 的 ThreadLocal 对象弱引用提供，所以这就有这种情况： 

如果 ThreadLocal 不设为 static 的，由于 Thread 的生命周期不可预知，这就导致了当系统 gc 时将会回收它，而 ThreadLocal 对象被回收了，此时它对应 key 必定为 null，这就导致了该 key 对应得 value 拿不出来了，而 value 之前被 Thread 所引用，所以就存在 key 为 null、value 存在强引用导致这个 Entry 回收不了，从而导致内存泄露。

所以避免内存泄露的方法，是对于 ThreadLocal 要设为 static 静态的，除了这个，还必须在线程不使用它的值是手动 remove 掉该 ThreadLocal 的值，这样 Entry 就能够在系统 gc 的时候正常回收，而关于 ThreadLocalMap 的回收，会在当前 Thread 销毁之后进行回收。

## **总结**

> 关于Volatile关键字具有可见性，但不具有操作的原子性，而synchronized比volatile对资源的消耗稍微大点，但可以保证变量操作的原子性，保证变量的一致性，最佳实践则是二者结合一起使用。

1、对于synchronized的出现，是解决多线程资源共享的问题，同步机制采用了“以时间换空间”的方式：访问串行化，对象共享化。同步机制是提供一份变量，让所有线程都可以访问。

2、对于Atomic的出现，是通过原子操作指令+Lock-Free完成，从而实现非阻塞式的并发问题。

3、对于Volatile，为多线程资源共享问题解决了部分需求，在非依赖自身的操作的情况下，对变量的改变将对任何线程可见。

4、对于ThreadLocal的出现，并不是解决多线程资源共享的问题，而是用来提供线程内的局部变量，省去参数传递这个不必要的麻烦，ThreadLocal采用了“以空间换时间”的方式：访问并行化，对象独享化。ThreadLocal是为每一个线程都提供了一份独有的变量，各个线程互不影响。



[volatile底层实现](https://my.oschina.net/u/2288283/blog/656572)














