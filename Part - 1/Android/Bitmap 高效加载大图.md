# Bitmap 高效加载大图

> java.lang.OutofMemoryError: bitmap size exceeds VM budget

Android 系统对每个应用使用的内存是有限制的，一旦应用使用的总内存超过这个阀值，系统就会抛出上面的错误导致应用 crash。根据经验来说，内存溢出很多场景都是由图片资源使用不当引起的，开发者文档中专门的[文章](https://developer.android.com/training/displaying-bitmaps/index.html)。

## 高效的加载高分辨率的图片

**问题：**

比如照片分辨率为2592x1936像素，如果我们使用ARGB_8888设置（android 2.3以后的默认设置，这种设置规定用四个字节来存储一个像素值）来加载这张图片，它将占用19M的内存（2592＊1936＊4字节）。在某些限制每个应用最多使用16M内存的手机上，这一张图片就会导致内存溢出。

**解决：**

其实在手机上展示图片时我们并不需要这么高分辨率的图片。比如在屏幕分辨率为1920x1080的手机上，即使你要展示图片的 imageview 充满了整个屏幕，也最多需要 1920x1080 的图片，更高分辨率的图片对我们的展示效果并没有提升，只会白白浪费我们宝贵的内存空间。所以对这种加载高分辨率的图片情况我们应该**加载一个低分辨率**版本的图片到内存中。

## Bitmap.Config

这是Bitmap的一个内部类(枚举)，是Bitmap关于色彩显示的配置，不同的配置对应不同的加载效果，下面是相应的文档介绍

它包含四个枚举值，分别如下

```java
Bitmap.Config {
  ALPHA_8
  ARGB_4444
  ARGB_8888
  RGB_565
}
```

具体这四个值分别代表了色彩的不同存储方式。这四个值指定了每一个像素的所占数据的大小，每个像素点都是1byte整数倍的数据。

- ALPHA_8代表每个像素点只占一个字节的大小，这个字节仅仅存储跟透明度相关的数据。
- ARGB_4444 代表16位ARGB位图
- ARGB_8888 代表32位ARGB位图，Android默认加载图片使用此种配置，所以每个像素占4字节。
- RGB_565 代表8位RGB位图

当然像素点占用的字节越多，他所存储的信息也就越多，图像也就越逼真，同时占用的内存也就越大。

## 大图缩放处理

设想一下自己手机拍了一张图片 大小为3120 * 4204，默认使用ARGB_8888的色彩存储方式，当把它加载到内存时，他的大小会是3120 * 4204 * 4 = 52465920字节 除以1024 * 1024 等于 50兆，通常的手机给每一个应用分配的内存大小，小点的也就16、32 兆左右， 大点的64、96M左右的样子。 这样的情况下，如果加载两张图片后，内存就不够了，此时虚拟机自动执行垃圾回收，因为图片很可能正在使用中，属于强引用， 此时是很难把图片占用的这部分内存回收掉， 所以如果按照原尺寸加载图片，很容易出现OOM。

加上我们实际需要显示的尺寸比真实图片尺寸小很多，所以，通常我们都会在图片显示时，对图片进行压缩处理，方法大都一样

### 1、读取位图的尺寸与类型

针对不同的图片数据来源，[BitmapFactory](http://developer.android.com/reference/android/graphics/BitmapFactory.html) 提供了一些解码（decode）的方法（[decodeByteArray()](http://developer.android.com/reference/android/graphics/BitmapFactory.html#decodeByteArray(byte[], int, int, android.graphics.BitmapFactory.Options)), [decodeFile()](http://developer.android.com/reference/android/graphics/BitmapFactory.html#decodeFile(java.lang.String, android.graphics.BitmapFactory.Options)), [decodeResource()](http://developer.android.com/reference/android/graphics/BitmapFactory.html#decodeResource(android.content.res.Resources, int, android.graphics.BitmapFactory.Options))等），这些方法在构造图片的时候会申请相应的内存空间，所以它们经常抛出内存溢出的异常。这些方法都允许传入一个 `BitmapFactory.Options` 类型的参数来获取将要构建的图片的属性。如果将 `inJustDecodeBounds` 的值设置成 `true`，这些方法将不会真正的创建图片，也就不会占用内存，它们的返回值将会是空。但是它们会读取图片资源的属性，但是可以获取到 outWidth, outHeight 与 outMimeType。我们就可以从 `BitmapFactory.Options` 中获取到图片的尺寸和类型。

```java
BitmapFactory.Options options = new BitmapFactory.Options();
//只加载图片边界信息到内存
options.inJustDecodeBounds = true;
BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
//获取图片的真实宽高
int imageHeight = options.outHeight;
int imageWidth = options.outWidth;
String imageType = options.outMimeType;
```

为了避免`java.lang.OutOfMemory` 的异常，我们需要在真正解析图片之前检查它的尺寸（除非你能确定这个数据源提供了准确无误的图片且不会导致占用过多的内存）。

### 2、根据给定尺寸，计算缩放比

第一步已获取图片的尺寸，接下来就是判断是将图片全尺寸加载到内存还是加载一个缩小版。判断的标准有以下几条：

* 全尺寸的图片需要占用多大内存空间；
* 应用能够提供给这张图片的内存空间的大小；
* 展示这张图片的 imageview 或其他 UI 组件的大小；
* 设备的屏幕大小和密度。

例如，填充 100x100 缩略图 imageview 时就没有必要将 1024x768 的图片全尺寸的加载到内存中。这时就要设置 `BitmapFactory.Options` 中的 `inSampleSize` 属性来告诉解码器来加载一个缩小比例的图片到内存中。如果以 `inSampleSize = 4` 的设置来解码这张 1024x768 的图片，将会得到一张 256x196 的图片，加载这张图片使用的内存会从 3M 减少到 196K 。`inSampleSize` 的值是根据缩放比例计算出来的一个 2 的 n 次幂的数。下面是计算 inSampleSize 常用的方法。

```java
public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // Raw height and width of image
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {

        final int halfHeight = height / 2;
        final int halfWidth = width / 2;

        // Calculate the largest inSampleSize value that is a power of 2 and keeps both
        // height and width larger than the requested height and width.
        while ((halfHeight / inSampleSize) > reqHeight
                && (halfWidth / inSampleSize) > reqWidth) {
            inSampleSize *= 2;
        }
    }
    return inSampleSize;
}
```

### 3、加载一个按比例缩小的版本到内存中

加载缩小比例图片大致有三个步骤：

1. `inJustDecodeBounds` 设置为 true 获取原始图片尺寸
2. 根据使用场景计算缩放比例 `inSampleSize` 
3. `inJustDecodeBounds` 设置为 false，传递 inSampleSize 给解码器创建缩小比例的图片。

```java
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
        int reqWidth, int reqHeight) {

    // First decode with inJustDecodeBounds=true to check dimensions
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);

    // Calculate inSampleSize
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

    // Decode bitmap with inSampleSize set
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}
```

有了上面的方法我们就可以很轻松的为大小为 100x100 缩略图 imageview 加载图片了

```java
mImageView.setImageBitmap(decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));
```

上述方法只针对一般意义上的大图，对于像长微博这种超大图片加载，仅仅依靠上述的方法是不能达到目的的，此时需要借助其他的 方法工具进行特殊处理,如局部加载等机制，具体可参考下面的两个链接。

[Android 高清加载巨图方案 拒绝压缩图片](http://blog.csdn.net/lmj623565791/article/details/49300989)

[WorldMap](https://github.com/johnnylambada/WorldMap)

## 参考资料

[Bitmap那些事之内存占用计算和加载注意事项](http://www.androidchina.net/2194.html)

[高效加载大图](http://hukai.me/android-training-course-in-chinese/graphics/displaying-bitmaps/load-bitmap.html)