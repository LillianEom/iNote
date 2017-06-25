# OkHttp3 用法全解析

http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0106/2275.html



### 异步 GET 请求

```
private void getAsynHttp() {
    mOkHttpClient = new OkHttpClient();
    Request.Builder requestBuilder = new Request.Builder().url("http://www.baidu.com");
    //可以省略，默认是GET请求
    requestBuilder.method("GET",null);
    Request request = requestBuilder.build();
    Call mcall= mOkHttpClient.newCall(request);
    mcall.enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {

        }

        @Override
        public void onResponse(Call call, Response response) throws IOException {
            if (null != response.cacheResponse()) {
                String str = response.cacheResponse().toString();
                Log.i("wangshu", "cache---" + str);
            } else {
                response.body().string();
                String str = response.networkResponse().toString();
                Log.i("wangshu", "network---" + str);
            }
            runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    Toast.makeText(getApplicationContext(), "请求成功", Toast.LENGTH_SHORT).show();
                }
            });
        }
    });
}
```

### 异步 POST 请求

OkHttp3 异步 POST 请求和 OkHttp2.x 有一些差别就是没有 FormEncodingBuilder 这个类，替代它的是功能更加强大的FormBody：

```
private void postAsynHttp() {
    mOkHttpClient=new OkHttpClient();
    RequestBody formBody = new FormBody.Builder()
            .add("size", "10")
            .build();
    Request request = new Request.Builder()
            .url("http://api.1-blog.com/biz/bizserver/article/list.do")
            .post(formBody)
            .build();
    Call call = mOkHttpClient.newCall(request);
    call.enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {

        }

        @Override
        public void onResponse(Call call, Response response) throws IOException {
            String str = response.body().string();
            Log.i("wangshu", str);

            runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    Toast.makeText(getApplicationContext(), "请求成功", Toast.LENGTH_SHORT).show();
                }
            });
        }

    });
}
```

#### POST 提交 Json 数据

```
public static final MediaType JSON = MediaType.parse("application/json; charset=utf-8");
OkHttpClient client = new OkHttpClient();
String post(String url, String json) throws IOException {
     RequestBody body = RequestBody.create(JSON, json);
      Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();
      Response response = client.newCall(request).execute();
    f (response.isSuccessful()) {
        return response.body().string();
    } else {
        throw new IOException("Unexpected code " + response);
    }
}
```

使用 Request 的 post 方法来提交请求体 RequestBody

#### POST 提交键值对

```
OkHttpClient client = new OkHttpClient();
String post(String url, String json) throws IOException {
 
     RequestBody formBody = new FormEncodingBuilder()
    .add("platform", "android")
    .add("name", "bug")
    .add("subject", "XXXXXXXXXXXXXXX")
    .build();
 
      Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();
 
      Response response = client.newCall(request).execute();
    if (response.isSuccessful()) {
        return response.body().string();
    } else {
        throw new IOException("Unexpected code " + response);
    }
}
```



### 异步上传文件

首先定义上传文件类型：

```
public static final MediaType MEDIA_TYPE_MARKDOWN 
					= MediaType.parse("text/x-markdown; charset=utf-8");
```

将 sdcard 根目录的 wangshu.txt 文件上传到服务器上：

```
 private void postAsynFile() {
    mOkHttpClient = new OkHttpClient();
    File file = new File("/sdcard/wangshu.txt");
    Request request = new Request.Builder()
            .url("https://api.github.com/markdown/raw")
            .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, file))
            .build();

    mOkHttpClient.newCall(request).enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {

        }

        @Override
        public void onResponse(Call call, Response response) throws IOException {
            Log.i("wangshu", response.body().string());
        }
    });
}
```

当然如果想要改为同步的上传文件只要调用 mOkHttpClient.newCall(request).execute()就可以了。

权限：

```
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

### 异步下载文件

下载一张图片，我们得到 Response 后将流写进我们指定的图片文件中就可以了。



```
private void downAsynFile() {
    mOkHttpClient = new OkHttpClient();
    String url = "http://img.my.csdn.net/uploads/201603/26/1458988468_5804.jpg";
    Request request = new Request.Builder().url(url).build();
    mOkHttpClient.newCall(request).enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {

        }
 
        @Override
        public void onResponse(Call call, Response response) {
            InputStream inputStream = response.body().byteStream();
            FileOutputStream fileOutputStream = null;
            try {
                fileOutputStream = new FileOutputStream(new File("/sdcard/wangshu.jpg"));
                byte[] buffer = new byte[2048];
                int len = 0;
                while ((len = inputStream.read(buffer)) != -1) {
                    fileOutputStream.write(buffer, 0, len);
                }
                fileOutputStream.flush();
            } catch (IOException e) {
                Log.i("wangshu", "IOException");
                e.printStackTrace();
           }

           Log.d("wangshu", "文件下载成功");
       }
   });
}
```

### 异步上传 Multipart 文件

这种场景很常用，我们有时会上传文件同时还需要传其他类型的字段，OkHttp3 实现起来很简单，需要注意的是没有服务器接收我这个 Multipart 文件，所以这里只是举个例子，具体的应用还要结合实际工作中对应的服务器。
首先定义上传文件类型：

```
private static final MediaType MEDIA_TYPE_PNG = MediaType.parse("image/png");
```

```
private void sendMultipart(){
    mOkHttpClient = new OkHttpClient();
    RequestBody requestBody = new MultipartBody.Builder()
            .setType(MultipartBody.FORM)
            .addFormDataPart("title", "wangshu")
            .addFormDataPart("image", "wangshu.jpg",
                    RequestBody.create(MEDIA_TYPE_PNG, new File("/sdcard/wangshu.jpg")))
            .build();
 
    Request request = new Request.Builder()
            .header("Authorization", "Client-ID " + "...")
            .url("https://api.imgur.com/3/image")
            .post(requestBody)
            .build();
 
   mOkHttpClient.newCall(request).enqueue(new Callback() {
       @Override
       public void onFailure(Call call, IOException e) {
 
       }
 
       @Override
       public void onResponse(Call call, Response response) throws IOException {
           Log.i("wangshu", response.body().string());
       }
   });
}
```

### 设置超时时间和缓存

和 OkHttp2.x 有区别的是不能通过 OkHttpClient 直接设置超时时间和缓存了，而是通过 OkHttpClient.Builder 来设置，通过 builder 配置好 OkHttpClient 后用 builder.build() 来返回 OkHttpClient，所以我们通常不会调用 new OkHttpClient() 来得到 OkHttpClient，而是通过 builder.build()：

```
File sdcache = getExternalCacheDir();
    int cacheSize = 10 * 1024 * 1024;
    OkHttpClient.Builder builder = new OkHttpClient.Builder()
            .connectTimeout(15, TimeUnit.SECONDS)
            .writeTimeout(20, TimeUnit.SECONDS)
            .readTimeout(20, TimeUnit.SECONDS)
            .cache(new Cache(sdcache.getAbsoluteFile(), cacheSize));
    OkHttpClient mOkHttpClient=builder.build();
```

### 取消请求和封装

```

```





