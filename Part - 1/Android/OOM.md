# Out Of Memory



## 发生OOM的条件

Android系统的每个进程都有一个最大内存限制，如果申请的内存资源超过这个限制，系统就会抛出OOM错误。

> - Android 2.x系统，当dalvik allocated + external allocated + 新分配的大小 >= dalvik heap 最大值时候就会发生OOM。其中bitmap是放于external中 。
> - Android 4.x系统，废除了external的计数器，类似bitmap的分配改到dalvik的java heap中申请，只要allocated + 新分配的内存 >= dalvik heap 最大值的时候就会发生OOM（art运行环境的统计规则还是和dalvik保持一致）

内存溢出是程序运行到某一阶段的最终结果，直接原因是剩余的内存不能满足内存的申请，但是再分析间接原因内存为什么没有了：

- 内存泄漏的存在可能导致可用内存越来越少；
- 内存申请的峰值超过了系统时间点剩余的内存；(例如：某手机单个进程可用最大内存为192M，目前分配内存80M，此时申请5M内存，但是当前时间点整个系统可用内存只有3M，此时没有超出单个进程可用最大内存，但是OOM也会发生)

## 避免 OOM

从四个方面着手，首先是减小对象的内存占用，其次是内存对象的重复利用，然后是避免对象的内存泄露，最后是内存使用策略优化

### 一、 减小对象的内存占用

避免OOM的第一步就是要尽量减少新分配出来的对象占用内存的大小，尽量使用更加轻量的对象。

#### 1）使用更加轻量的数据结构

可以考虑使用ArrayMap/SparseArray而不是HashMap等传统数据结构。通常的HashMap的实现方式更加消耗内存，因为它需要一个额外的实例对象来记录Mapping操作。另外，SparseArray更加高效在于他们避免了对key与value的autobox自动装箱，并且避免了装箱后的解箱。

#### 2）避免在Android里面使用Enum

比static变量占用内存多几倍

#### 3）减小Bitmap对象的内存占用

Bitmap是一个极容易消耗内存的大胖子，减小创建出来的Bitmap的内存占用是很重要的，通常来说有下面2个措施：

> inSampleSize：缩放比例，在把图片载入内存之前，我们需要先计算出一个合适的缩放比例，避免不必要的大图载入。decode format：解码格式，选择ARGB_8888/RBG_565/ARGB_4444/ALPHA_8，存在很大差异。

* **在内存中压缩图片**

  装载大图片时需要对图片进行压缩，使用等比例压缩的方法直接在内存中处理图片

  ```java
  Options options = new BitmapFactory.Options();
  options.inSampleSize = 5; // 原图的五分之一，设置为2则为二分之一
  BitmapFactory.decodeFile(myImage.getAbsolutePath(), options);
  ```

  这样做要注意的是，图片质量会变差，inSampleSize设置的值越大，图片质量就越差，不同的手机厂商缩放的比例可能不同。

* **使用完图片后回收图片所占内存**

  ```java
  if (!bitmapObject.isRecyled()) {     // Bitmap对象没有被回收
       bitmapObject.recycle();     // 释放
       System.gc();     // 提醒系统及时回收
  }
  ```


* **降低要显示的图片色彩质量**

  Android中Bitmap有四种图片色彩模式：

  ALPHA_8：每个像素需要占用内存中的1byte_

  _RGB_565：每个像素需要占用内存中的2byte

  ARGB_4444：每个像素需要占用内存中的2byte_

  _ARGB_8888：每个像素需要占用内存中的4byte

  我们创建Bitmap时，默认的色彩模式是ARGB_8888的，这种色彩模式是质量最高的，当然这样的模式占用的内存也最大。_

  而ARGB_4444每个像素只占用2byte，所以使用ARGB_4444的模式也能降低图片占用的内存大小。

  ```java
  BitmapFactory.Options options = new BitmapFactory.Options();
  options.inPreferredConfig = Bitmap.Config.ARGB_4444;
  Bitmap btimapObject = BitmapFactory.decodeFile(myImage.getAbsolutePath(),
  ```

  其实大多数图片设置成ARGB_4444模式后，在显示上是看不出与ARGB_8888模式有什么差别的，只是在具有渐变色效果的图片时，可能会让渐变色呈现色彩条样的效果。

  这种降低色彩质量的方法对内存的降低效果不如方法1明显。

* **查询图片信息时不把图片加载到内存中**

  有时候我们取得一张图片，也许只是为了获得这个图片的一些信息，比如图片的width、height等信息，不需要显示到界面上，这个时候我们可以不把图片加载到内存中。

  ```java
  BitmapFactory.Options options = new BitmapFactory.Options();
  options.inJustDecodeBounds = true;     // 不把图片加载到内存中
  Bitmap btimapObject = BitmapFactory.decodeFile(myImage.getAbsolutePath(), options
  ```

  inJustDecodeBounds属性，如果值为true，那么将不返回实际的Bitmap对象，也不给其分配内存空间，但允许我们查询图片宽、高、大小等基本信息。

  （获取原始宽高：options.outWidth，options.outHeight）

#### 4）使用更小的图片

在设计给到资源图片的时候，我们需要特别留意这张图片是否存在可以压缩的空间，是否可以使用一张更小的图片。尽量使用更小的图片不仅仅可以减少内存的使用，还可以避免出现大量的InflationException。假设有一张很大的图片被XML文件直接引用，很有可能在初始化视图的时候就会因为内存不足而发生InflationException，这个问题的根本原因其实是发生了OOM。

### 二、内存对象的重复利用

大多数对象的复用，最终实施的方案都是利用对象池技术，要么是在编写代码的时候显式的在程序里面去创建对象池，然后处理好复用的实现逻辑，要么就是利用系统框架既有的某些复用特性达到减少对象的重复创建，从而减少内存的分配与回收。在Android上面最常用的一个缓存算法是LRU(Least Recently Use)

#### 1）复用系统自带的资源

Android系统本身内置了很多的资源，例如字符串/颜色/图片/动画/样式以及简单布局等等，这些资源都可以在应用程序中直接引用。这样做不仅仅可以减少应用程序的自身负重，减小APK的大小，另外还可以一定程度上减少内存的开销，复用性更好。但是也有必要留意Android系统的版本差异性，对那些不同系统版本上表现存在很大差异，不符合需求的情况，还是需要应用程序自身内置进去。

#### 2）注意在ListView/GridView等出现大量重复子组件的视图里面对ConvertView的复用

#### 3）Bitmap对象的复用

- 在ListView与GridView等显示大量图片的控件里面需要使用**LRU**的机制来缓存处理好的Bitmap。


- 利用inBitmap的高级特性提高Android系统在Bitmap分配与释放执行效率上的提升(3.0以及4.4以后存在一些使用限制上的差异)。使用inBitmap属性可以告知Bitmap解码器去尝试使用已经存在的内存区域，新解码的bitmap会尝试去使用之前那张bitmap在heap中所占据的pixel data内存区域，而不是去问内存重新申请一块区域来存放bitmap。利用这种特性，即使是上千张的图片，也只会仅仅只需要占用屏幕所能够显示的图片数量的内存大小。


- 在SDK 11 -> 18之间，重用的bitmap大小必须是一致的，例如给inBitmap赋值的图片大小为100-100，那么新申请的bitmap必须也为100-100才能够被重用。从SDK 19开始，新申请的bitmap大小必须小于或者等于已经赋值过的bitmap大小。
- 新申请的bitmap与旧的bitmap必须有相同的解码格式，例如大家都是8888的，如果前面的bitmap是8888，那么就不能支持4444与565格式的bitmap了。 我们可以创建一个包含多种典型可重用bitmap的对象池，这样后续的bitmap创建都能够找到合适的“模板”去进行重用。如下图所示：

![android_perf_2_inbitmap_pool](http://hukai.me/images/android_perf_2_inbitmap_pool.png)

另外提一点：在2.x的系统上，尽管bitmap是分配在native层，但是还是无法避免被计算到OOM的引用计数器里面。这里提示一下，不少应用会通过反射BitmapFactory.Options里面的**inNativeAlloc**来达到扩大使用内存的目的，但是如果大家都这么做，对系统整体会造成一定的负面影响，建议谨慎采纳。

#### 4）避免在onDraw方法里面执行对象的创建

类似onDraw等频繁调用的方法，一定需要注意避免在这里做创建对象的操作，因为他会迅速增加内存的使用，而且很容易引起频繁的gc，甚至是内存抖动。

#### 5）StringBuilder

在有些时候，代码中会需要使用到大量的字符串拼接的操作，这种时候有必要考虑使用StringBuilder来替代频繁的“+”。

### 三、避免对象的内存泄露





[Android内存优化之OOM--胡凯](http://hukai.me/android-performance-oom/)

[Android 内存优化总结&实践](http://mp.weixin.qq.com/s/2MsEAR9pQfMr1Sfs7cPdWQ)













