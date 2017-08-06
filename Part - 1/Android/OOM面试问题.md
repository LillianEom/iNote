# OOM系列问题

### 1. 什么OOM？

OOM 全称是 Out Of Merrory，Android系统的每一个应用程序都设置一个硬性的Dalvik Heap Size最大限制阈值，如果申请的内存资源超过这个限制，系统就会抛出OOM错误

### 2. 内存泄漏有哪些场景以及解决方法

  * **类的静态变量持有大数据对象** 

    静态变量长期维持到大数据对象的引用，阻止垃圾回收。

  * **非静态内部类存在静态实例** 

    非静态内部类会维持一个到外部类实例的引用，如果非静态内部类的实例是静态的，就会间接长期维持着外部类的引用，阻止被回收掉。

  * **资源对象未关闭**

     资源性对象比如（Cursor，File文件等）往往都用了一些缓冲，我们在不使用的时候，应该及时关闭它们， 以便它们的缓冲及时回收内存。它们的缓冲不仅存在于java虚拟机内，还存在于java虚拟机外。 如果我们仅仅是把它的引用设置为null,而不关闭它们，往往会造成内存泄露。

    解决办法： 比如SQLiteCursor（在析构函数finalize（）,如果我们没有关闭它，它自己会调close()关闭）， 如果我们没有关闭它，系统在回收它时也会关闭它，但是这样的效率太低了。 因此对于资源性对象在不使用的时候，应该调用它的close()函数，将其关闭掉，然后才置为null. 在我们的程序退出时一定要确保我们的资源性对象已经关闭。 程序中经常会进行查询数据库的操作，但是经常会有使用完毕Cursor后没有关闭的情况。如果我们的查询结果集比较小，对内存的消耗不容易被发现，只有在常时间大量操作的情况下才会复现内存问题，这样就会给以后的测试和问题排查带来困难和风险，记得try catch后，在finally方法中关闭连接

  * **Handler内存泄漏**

    Handler作为内部类存在于Activity中，但是Handler生命周期与Activity生命周期往往并不是相同的，比如当Handler对象有Message在排队，则无法释放，进而导致本该释放的Acitivity也没有办法进行回收。 解决办法：

    声明handler为static类，这样内部类就不再持有外部类的引用了，就不会阻塞Activity的释放

  * 如果内部类实在需要用到外部类的对象，可在其内部声明一个弱引用引用外部类。
  ​

```java
public class MainActivity extends Activity {
  private CustomHandler mHandler;
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mHandler = new CustomHandler(this);
}
```


```java
static class CustomHandler extends Handler {
  // 内部声明一个弱引用，引用外部类
  private WeakReference<MainActivity > activityWeakReference;
  public MyHandler(MyActivity activity) {
    activityWeakReference= new WeakReference<MainActivity >(activity);
  }
  // ... ...   
}

}
//在Activity onStop或者onDestroy的时候，取消掉该Handler对象的Message和Runnable
  
  @Override
  public void onDestroy() {
  //  If null, all callbacks and messages will be removed.
  mHandler.removeCallbacksAndMessages(null);
}
```
一些不良代码习惯 有些代码并不造成内存泄露，但是他们的资源没有得到重用，频繁的申请内存和销毁内存，消耗CPU资源的同时，也引起内存抖动 解决方案 如果需要频繁的申请内存对象和和释放对象，可以考虑使用**对象池来增加对象的复用**。 例如ListView便是采用这种思想，通过复用converview来避免频繁的GC





### 如何避免 OOM 问题的出现

1. **使用更加轻量的数据结构**

   例如，我们可以考虑使用ArrayMap/SparseArray而不是HashMap等传统数据结构。通常的HashMap的实现方式更加消耗内存，因为它需要一个额外的实例对象来记录Mapping操作。另外，SparseArray更加高效，在于他们避免了对key与value的自动装箱（autoboxing），并且避免了装箱后的解箱。


2. **避免在Android里面使用Enum** 

   Android官方培训课程提到过“Enums often require more than twice as much memory as static constants. You should strictly avoid using enums on Android.”，具体原理请参考《Android性能优化典范（三）》，所以请避免在Android里面使用到枚举。

3. **减小Bitmap对象的内存占用** 

  Bitmap是一个极容易消耗内存的大胖子，减小创建出来的Bitmap的内存占用可谓是重中之重，，通常来说有以下2个措施： **inSampleSize：**缩放比例，在把图片载入内存之前，我们需要先计算出一个合适的缩放比例，避免不必要的大图载入。 decode format：解码格式，选择ARGB_6666/RBG_545/ARGB_4444/ALPHA_6，存在很大差异

  **Bitmap对象的复用** 缩小Bitmap的同时，也需要提高BitMap对象的复用率，避免频繁创建BitMap对象，复用的方法有以下2个措施 **LRUCache :** “最近最少使用算法”在Android中有极其普遍的应用。ListView与GridView等显示大量图片的控件里，就是使用LRU的机制来缓存处理好的Bitmap，把近期最少使用的数据从缓存中移除，保留使用最频繁的数据， inBitMap高级特性:利用inBitmap的高级特性提高Android系统在Bitmap分配与释放执行效率。使用inBitmap属性可以告知Bitmap解码器去尝试使用已经存在的内存区域，新解码的Bitmap会尝试去使用之前那张Bitmap在Heap中所占据的pixel data内存区域，而不是去问内存重新申请一块区域来存放Bitmap。利用这种特性，即使是上千张的图片，也只会仅仅只需要占用屏幕所能够显示的图片数量的内存大小

4. **使用更小的图片** 

  在涉及给到资源图片时，我们需要特别留意这张图片是否存在可以压缩的空间，是否可以使用更小的图片。尽量使用更小的图片不仅可以减少内存的使用，还能避免出现大量的InflationException。假设有一张很大的图片被XML文件直接引用，很有可能在初始化视图时会因为内存不足而发生InflationException，这个问题的根本原因其实是发生了OOM。

5. **StringBuilder** 在有些时候，代码中会需要使用到大量的字符串拼接的操作，这种时候有必要考虑使用StringBuilder来替代频繁的“+”。

6. **避免在onDraw方法里面执行对象的创建**

  类似onDraw等频繁调用的方法，一定需要注意避免在这里做创建对象的操作，因为他会迅速增加内存的使用，而且很容易引起频繁的gc，甚至是内存抖动。

7. **避免对象的内存泄露** 

   android中内存泄漏的场景以及解决办法，参考上一问