# DiskLruCache 磁盘缓存

使用缓存策略， 对网络上下载的图片等资源文件进行缓存， 当再次请求同一个资源 url 时， 首先从缓存中查找是否存在， 当不存在时再从网络上下载。采用缓存，除了提高获取资源的速度，也对减少使用用户手机上的流量有很好的作用. 核心思想是当缓存满时，会优先淘汰那些最少使用的缓存对象。采用 LRU 算法的缓存有两种，LruCache 用于内存缓存， DiskLruCache 用于存储设备缓存, 它通过把对象写入文件系统从而实现缓存的效果.

LruCache 和 DiskLruCache 的区别：

- 共同点：两者都是缓存信息，并且都是采用LRU算法。
- 不同点：LruCache读写速度快，但存储大小有限。DiskLruCache读写速度慢，但存储空间大。



## 创建

```java
private static final long DISK_CACHE_SIZE = 1024 * 1024 * 50;// 50MB

File diskCacheDir = new File(mContext,"bitmap");
if(!diskCacheDir.exists()){
   diskCacheDir.mkdirs();
}
mDiskLruCache = DiskLruCache.open(diskCacheDir,1,1,DISK_CACHE_SIZE);
```

- `diskCacheDir` 缓存文件夹，具体指`sdcard/Android/data/package_name/cache`
- `1` 应用版本号，一般写为1
- `1` 单个节点所对应的数据的个数，一般写1
- `DISK_CACHE_SIZE` 缓存大小

------

## 缓存添加

> DishLruCache 缓存添加的操作通过 Eidtor 完成，Editor 为一个缓存对象的编辑对象。

首先需要获取图片的`url`所对应的`key`，根据`key`利用`edit()`来获取`Editor`对象。若此时，这个缓存正在被编辑，`edit()`会返回`null`。`DiskLruCache`不允许同时编辑同一个缓存对象。之所以把`url`转换成`key`，因为图片的`url`中可能存在特殊字符，会影响使用，一般将`url`的`md5`值作为`key`

归纳来说添加操作分为三步：

- 通过文件的Url将文件写入文件系统
- 通过Url对应的key来得到一个不为空的Editor对象
- 通过这个Editor对象来对写入操作进行提交或者撤销操作

  ​

```java
private String hashKeyFromUrl(String url){
    String cacheKey;
    try {
        final MessageDigest  mDigest = MessageDigest.getInstance("MD5");
        mDigest.update(url.getBytes());
        cacheKey = byteToHexString(mDigest.digest());
    } catch (NoSuchAlgorithmException e) {
        cacheKey = String.valueOf(url.hashCode());
    }
     return  cacheKey;
}

private String byteToHexString(byte[] bytes) {
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < bytes.length; i ++){
          String hex = Integer.toHexString(0xFF & bytes[i]);//得到十六进制字符串
          if (hex.length() == 1){
             sb.append('0');
          }
          sb.append(hex);
    }
    return  sb.toString();
}
```

将`url`转成`key`，利用这`key`值获取`Editor`对象。若这个`key`的`Editor`对象不存在，`edit()`方法就创建一个新的出来。通过`Editor`对象可以获取一个输出流对象。`DiskLruCache`的`open()`方法中，一个节点只能有一个数据，`edit.newOutputStream(DISK_CACHE_INDEX)`参数设置为`0`

```java
String key = hashKeyFromUrl(url);
DiskLruCache.Editor editor =mDiskLruCache.edit(key);
if (editor != null){
    OutputStream outputStream = editor.newOutputStream(DISK_CACHE_INDEX);
}
```

有了这个文件输出流，从网络加载一个图片后，通过这个`OutputStream outputStream`写入文件系统

```java
private boolean downLoadUrlToStream(String urlString, OutputStream outputStream) {
        HttpURLConnection urlConnection = null;
        BufferedOutputStream bos = null;
        BufferedInputStream bis = null;
        try {
            final URL url   = new URL(urlString);
            urlConnection = (HttpURLConnection) url.openConnection();
            bis = new BufferedInputStream(urlConnection.getInputStream(),8 * 1024);
            bos = new BufferedOutputStream(outputStream,8 * 1024);
            int b ;
            while((b = bis.read())!= -1){
                bos.write(b);
            }
            return  true;
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            if (urlConnection != null){
                urlConnection.disconnect();
            }
            closeIn(bis) ;
            closeOut(bos);
        }
        return   false;
}
```

上面的代码并没有将图片写入文件系统，还需要通过`Editor.commit()`提交写入操作，若写入失败，调用`abort()`方法，进行回退整个操作

```java
if (downLoadUrlToStream(url,outputStream)){
    editor.commit();//提交
}else {
    editor.abort();//重复操作
}
```

这时，图片已经正确写入文件系统，接下来的图片获取就不需要请求网络

------

## 缓存查找

> * 查找过程，也需要将`url`转换为`key`
> * 然后通过`DiskLruCache`的`get`方法得到一个`Snapshot`对象
> * 再通过`Snapshot`对象可得到缓存的文件输入流
> * 有了输入流就可以得到`Bitmap`对象

为了避免`oom`，会使用`ImageResizer`进行缩放。若直接对`FileInputStream`进行操作，缩放会出现问题。`FileInputStream`是有序的文件流，两次`decodeStream`调用会影响文件流的位置属性。可以通过文件流得到其所对应的文件描述符，利用`BitmapFactory.decodeFileDescriptor()`方法进行缩放

```java
Bitmap bitmap = null;
String key = hashKeyFromUrl(url);
DiskLruCache.Snapshot snapshot = mDiskLruCache.get(key);
if (snapshot != null){
    FileInputStream fis = (FileInputStream) snapshot.getInputStream(DISK_CACHE_INDEX);
    FileDescriptor fileDescriptor = fis.getFD();
    bitmap = imageResizer.decodeBitmapFromFileDescriptor(fileDescriptor,targetWidth,targetHeight);
    if (bitmap != null){
        addBitmapToMemoryCache(key,bitmap);
    }
}
```

在查找得到`Bitmap`后，把`key,bitmap`添加到内存缓存中

## ImageLoader的实现

主要思路：

1. 拿到图片请求地址`url`后，先把`url`变作对应的`key`；
2. 利用`key`在内存缓存中查找，查找到了就进行加载显示图片；若没有查到就进行`3`
3. 在磁盘缓存中查找，在若查到了，就加载到内存缓存，后加载显示图片；若没有查找到，进行`4`
4. 进行网络求，拿到数据图片，把图片写进磁盘缓存成功后，再加入到内存缓存中，并根据实际所需显示大小进行合理缩放显示





项目地址:

[DiskLruCache-JakeWharton实现版](https://github.com/JakeWharton/DiskLruCache)

[DiskLruCache - Android官方文档推荐](https://android.googlesource.com/platform/libcore/+/android-4.1.1_r1/luni/src/main/java/libcore/io/DiskLruCache.java)

[DiskLruCache源码分析](http://www.jianshu.com/p/b282140acc20)

[Android：跟着实战项目学缓存策略之DiskLruCache详谈](http://www.jianshu.com/p/4320597ebd7e)



