# OKHttp 缓存策略



## Http 缓存机制

缓存的好处

- 减少请求次数，减小服务器压力.
- 本地数据读取速度更快，让页面不会空白几百毫秒。
- 在无网络的情况下提供数据。

缓存一般由服务器控制(通过某些方式可以本地控制缓存，比如向过滤器添加缓存控制信息)。

### 缓存 Header 的含义

* Expires

  缓存过期的时间（绝对时间），一般用在 response 报文中，当超过此事件后响应将被认为是无效的而需要网络连接，反之直接使用缓存

  > Expires: Thu, 12 Jan 2017 11:01:33 GMT


* Cache-Control

  由服务器返回的 Response 中添加的头信息，它的目的是告诉客户端是要从本地读取缓存还是直接从服务器摘取消息。它有不同的值，每一个值有不同的作用。


* 修订文件名(Reving Filenames)

  如果我们通过设置header保证了客户端可以缓存的，而此时远程服务器更新了文件如何解决呢？我们这时可以通过修改url中的文件名版本后缀进行缓存，比如下文是又拍云的公共CDN就提供了多个版本的JQuery

  > upcdn.b0.upaiyun.com/libs/jquery/jquery-2.0.3.min.js


* 条件 GET 请求 (Conditional GET Requests) 与 304

  如缓存果过期或者强制放弃缓存，在此情况下，缓存策略全部交给服务器判断，客户端只用发送条件 `get` 请求即可，如果缓存是有效的，则返回` 304 Not Modifiled`，否则直接返回body。

  请求的方式有两种：

  * Last-Modified-Date:

    客户端第一次网络请求时，服务器返回了

    > Last-Modified: Tue, 12 Jan 2016 09:31:27 GMT

    客户端再次请求时，通过发送

    > If-Modified-Since: Tue, 12 Jan 2016 09:31:27 GMT

    交给服务器进行判断，如果仍然可以缓存使用，服务器就返回 `304`

  *  ETag

    ETag是对资源文件的一种摘要，客户端并不需要了解实现细节。当客户端第一请求时，服务器返回了

    > ETag: "5694c7ef-24dc"

    客户端再次请求时，通过发送

    > If-None-Match:"5694c7ef-24dc"

    交给服务器进行判断，如果仍然可以缓存使用，服务器就返回304

如果 ETag 和 Last-Modified 都有，则必须一次性都发给服务器，它们没有优先级之分，反正这里客户端没有任何判断的逻辑。

* 其它标签

  no-cache/no-store: 不使用缓存，no-cache 指令的目的是防止从缓存中返回过期的资源。客户端发送的请求中如果包含 no-cache 指令的话，表示客户端将不会接受缓存过的相应，于是缓存服务器必须把客户端请求转发给源服务器。服务器端返回的相应中包含 no-cache 指令的话那么缓存服务器不能对资源进行缓存。

![](http://om4rextnc.bkt.clouddn.com/17-7-13/71584547.jpg)

<div class="tip">

注意服务器返回 304 意思是数据没有变动滚去读缓存信息。

</div>

## OkHttp 缓存机制

OkHttp 中使用了 [`CacheStrategy`](https://github.com/square/okhttp/blob/db9c2db40b0b89a1853715fd52e2748463d9cc9c/okhttp/src/main/java/okhttp3/internal/http/CacheStrategy.java#L31-L31) 实现了上文的流程图，它根据之前的缓存结果与当前将要发送 Request 的 header 进行策略分析，并得出是否进行请求的结论。

### 总体请求流程

CacheStrategy 类似一个`mapping`操作，将两个值输入，再将两个值输出

| Input         | request, cacheCandidate       |
| ------------- | ----------------------------- |
| ↓             | ↓                             |
| CacheStrategy | 处理，判断Header信息                 |
| ↓             | ↓                             |
| Output        | networkRequest, cacheResponse |

Request:

开发者手动编写并在 Interceptor 中递归加工而成的对象，我们只需要知道了目前传入的 Request 中并没有任何关于缓存的 Header

cacheCandidate:

就是上次与服务器交互缓存的 Response，可能为 null。这里的缓存全部是基于文件系统的 Map，key 是请求中 url 的 md5，value 是在文件中查询到的缓存，页面置换基于 LRU 算法，我们现在只需要知道它是一个可以读取缓存 Header 的 Response 即可。

当被 CacheStrategy 加工输出后，输出 networkRequest 与 cacheResponse，根据是否为空执行不同的请求

| networkRequest | cacheResponse | result                                   |
| -------------- | ------------- | ---------------------------------------- |
| null           | null          | only-if-cached(表明不进行网络请求，且缓存不存在或者过期，一定会返回503错误) |
| null           | non-null      | 不进行网络请求，而且缓存可以使用，直接返回缓存，不用请求网络           |
| non-null       | null          | 需要进行网络请求，而且缓存不存在或者过期，直接访问网络              |
| non-null       | non-null      | Header中含有ETag/Last-Modified标签，需要在条件请求下使用，还是需要访问网络 |

### CacheStrategy 的加工过程

`CacheStrategy`使用[Factory模式](https://en.wikipedia.org/wiki/Factory_method_pattern)进行构造，参数如下

```java
nternalCache responseCache = Internal.instance.internalCache(client);
//cacheCandidate从disklurcache中获取
//request的url被md5序列化为key,进行缓存查询
Response cacheCandidate = responseCache != null ? responseCache.get(request) : null;
//请求与缓存
factory = new CacheStrategy.Factory(now, request, cacheCandidate);
cacheStrategy = factory.get();
//输出结果
networkRequest = cacheStrategy.networkRequest;
cacheResponse = cacheStrategy.cacheResponse;
//进行一大堆的if判断，内容同上表格
.....
```

可以看出`Factory.get()`是最关键的缓存策略的判断，我们点入`get()`[方法](https://github.com/square/okhttp/blob/db9c2db40b0b89a1853715fd52e2748463d9cc9c/okhttp/src/main/java/okhttp3/internal/http/CacheStrategy.java#L166-L166)，可以发现是对`getCandidate()`的一个封装，我们接着点开`getCandidate()`，全是if与数学计算，详细代码如下

```java
private CacheStrategy getCandidate() {
  //如果缓存没有命中(即null),网络请求也不需要加缓存Header了
  if (cacheResponse == null) {
    //`没有缓存的网络请求,查上文的表可知是直接访问
    return new CacheStrategy(request, null);
  }

  // 如果缓存的TLS握手信息丢失,返回进行直接连接
  if (request.isHttps() && cacheResponse.handshake() == null) {
    //直接访问
    return new CacheStrategy(request, null);
  }

  //检测response的状态码,Expired时间,是否有no-cache标签
  if (!isCacheable(cacheResponse, request)) {
    //直接访问
    return new CacheStrategy(request, null);
  }

  CacheControl requestCaching = request.cacheControl();
  //如果请求报文使用了`no-cache`标签(这个只可能是开发者故意添加的)
  //或者有ETag/Since标签(也就是条件GET请求)
  if (requestCaching.noCache() || hasConditions(request)) {
    //直接连接,把缓存判断交给服务器
    return new CacheStrategy(request, null);
  }
  //根据RFC协议计算
  //计算当前age的时间戳
  //now - sent + age (s)
  long ageMillis = cacheResponseAge();
  //大部分情况服务器设置为max-age
  long freshMillis = computeFreshnessLifetime();

  if (requestCaching.maxAgeSeconds() != -1) {
    //大部分情况下是取max-age
    freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
  }

  long minFreshMillis = 0;
  if (requestCaching.minFreshSeconds() != -1) {
    //大部分情况下设置是0
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
  }

  long maxStaleMillis = 0;
  //ParseHeader中的缓存控制信息
  CacheControl responseCaching = cacheResponse.cacheControl();
  if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
    //设置最大过期时间,一般设置为0
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
  }

  //缓存在过期时间内,可以使用
  //大部分情况下是进行如下判断
  //now - sent + age + 0 < max-age + 0
  if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
    //返回上次的缓存
    Response.Builder builder = cacheResponse.newBuilder();
    return new CacheStrategy(null, builder.build());
  }

  //缓存失效, 如果有etag等信息
  //进行发送`conditional`请求,交给服务器处理
  Request.Builder conditionalRequestBuilder = request.newBuilder();

  if (etag != null) {
    conditionalRequestBuilder.header("If-None-Match", etag);
  } else if (lastModified != null) {
    conditionalRequestBuilder.header("If-Modified-Since", lastModifiedString);
  } else if (servedDate != null) {
    conditionalRequestBuilder.header("If-Modified-Since", servedDateString);
  }
  //下面请求实质还说网络请求
  Request conditionalRequest = conditionalRequestBuilder.build();
  return hasConditions(conditionalRequest) ? new CacheStrategy(conditionalRequest,
      cacheResponse) : new CacheStrategy(conditionalRequest, null);
}
```

通过上面的分析，okhttp 实现的缓存策略实质上就是大量的 if 判断集合。Okhttp 的缓存是自动完成的，完全由服务器Header决定的。

## Cache的实现方式

通过上面的分析可见**控制缓存的消息头**往往是服务端返回的信息中添加的如**”Cache-Control:max-age=60”**。所以，会有两种情况：

* 客户端和服务端开发能够很好沟通，按照达成一致的协议，只要服务器在返回消息的时候添加好 Cache-Control 相关的消息便好。
* 客户端与服务端的开发根本就不是同一家公司，没有办法也不可能要求服务端按照客户端的意愿进行开发。

那就是定义一个拦截器，人为地添加 Response 中的消息头，然后再传递给用户，这样用户拿到的 Response 就有了我们理想当中的消息头Headers，从而达到控制缓存的意图。

### 缓存之拦截器

因为拦截器可以拿到 Request 和 Response，人为地添加 Cache-Control 消息头。

```java
class CacheInterceptor implements Interceptor{
        @Override
        public Response intercept(Chain chain) throws IOException {

            Response originResponse = chain.proceed(chain.request());

            //设置缓存时间为60秒，并移除了pragma消息头，移除它的原因是因为pragma也是控制缓存的一个消息头属性
            return originResponse.newBuilder().removeHeader("pragma")
                    .header("Cache-Control","max-age=60").build();
        }
}
```

定义好拦截器中后，我们可以添加到 OKHttpClient 中了。

```java
private void testCacheInterceptor(){
        //缓存文件夹
        File cacheFile = new File(getExternalCacheDir().toString(),"cache");
        //缓存大小为10M
        int cacheSize = 10 * 1024 * 1024;
        //创建缓存对象
        final Cache cache = new Cache(cacheFile,cacheSize);

        OkHttpClient client = new OkHttpClient.Builder()
                .addNetworkInterceptor(new CacheInterceptor())
                .cache(cache)
                .build();
        .......
}
```

#### 拦截器进行缓存的缺点

情况：

1. 网络访问请求的资源是文本信息，如新闻列表，这类信息经常变动，一天更新好几次，它们用的**缓存时间应该就很短**。 
2. 网络访问请求的资源是图片或者视频，它们变动很少，或者是长期不变动，那么它们用的**缓存时间就应该很长**。

同一个 APP，用同一个 OKHTTPCLIENT 对象这是为了只有一个缓存文件访问入口。这个很容易理解，单例模式嘛。但是拦截器是在 OKHttpClient.Builder 当中添加的。如果在拦截器中定义缓存的方法会导致图片的缓存和新闻列表的缓存时间是一样的，这显然是不合理的。真实的情况不应该是图片请求有它的缓存时间，新闻列表请求有它的缓存时间，应该是每一个 Request 有它的缓存时间。 

解决这个问题可以用okhttp 官方建议的方法。

### OkHttp 官方文档建议缓存方法

OkHttp 的官方缓存控制在[注释中](https://github.com/square/okhttp/blob/d662c1a82851800c46ad8ede2d9d10d10427fdad/okhttp/src/main/java/okhttp3/Cache.java#L79)

OkHttp 中建议用 CacheControl 这个类来进行缓存策略的制定。 
它内部有两个很重要的静态实例。

```java
/**强制使用网络请求*/
public static final CacheControl FORCE_NETWORK = new Builder().noCache().build();
  /**
   * 强制性使用本地缓存，如果本地缓存不满足条件，则会返回code为504
   */
public static final CacheControl FORCE_CACHE = new Builder()
     .onlyIfCached()
     .maxStale(Integer.MAX_VALUE, TimeUnit.SECONDS)
     .build();
```

`FORCE_NETWORK` 常量用来强制使用网络请求。`FORCE_CACHE` 只取本地的缓存。它们本身都是 CacheControl 对象，由内部的 Buidler 对象构造。

#### CacheControl.Builder

它有如下方法：

```java
- noCache();//不使用缓存，用网络请求
- noStore();//不使用缓存，也不存储缓存
- onlyIfCached();//只使用缓存
- noTransform();//禁止转码
- maxAge(10, TimeUnit.MILLISECONDS);//设置超时时间为10ms。
- maxStale(10, TimeUnit.SECONDS);//超时之外的超时时间为10s
- minFresh(10, TimeUnit.SECONDS);//超时时间为当前时间加上10秒钟。
```

知道了 CacheControl 的相关信息，那么它怎么使用呢？不同于拦截器设置缓存，**CacheControl 是针对 Request 的**，所以它可以针对每个请求设置不同的缓存策略。比如图片和新闻列表。下面代码展示如何用 CacheControl 设置一个 60 秒的超时时间。

```java
private void testCacheControl(){
    //缓存文件夹
    File cacheFile = new File(getExternalCacheDir().toString(),"cache");
    //缓存大小为10M
    int cacheSize = 10 * 1024 * 1024;
    //创建缓存对象
    final Cache cache = new Cache(cacheFile,cacheSize);

    new Thread(new Runnable() {
        @Override
        public void run() {
            OkHttpClient client = new OkHttpClient.Builder()
                    .cache(cache)
                    .build();
            //设置缓存时间为60秒
            CacheControl cacheControl = new CacheControl.Builder()
                    .maxAge(60, TimeUnit.SECONDS)
                    .build();
            Request request = new Request.Builder()
                    .url("http://blog.csdn.net/briblue")
                    .cacheControl(cacheControl)
                    .build();

            try {
                Response response = client.newCall(request).execute();

                response.body().close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }).start();
}
```

#### 强制使用缓存

`CacheControl.FORCE_CACHE` 常量

```java
public static final CacheControl FORCE_CACHE = new Builder()
      .onlyIfCached()
      .maxStale(Integer.MAX_VALUE, TimeUnit.SECONDS)
      .build();
```

它内部其实就是调用 `onlyIfCached()` 和 `maxStale` 方法。 
它的使用方法为

```java
Request request = new Request.Builder()
            .url("http://blog.csdn.net/briblue")
            .cacheControl(Cache.FORCE_CACHE)
            .build();
```

#### 不使用缓存

`CacheControl.FORCE_NETWORK` 常量

```java
public static final CacheControl FORCE_NETWORK = new Builder().noCache().build();
```

它的内部其实是调用 `noCache()` 方法，也就是不缓存的意思。 
它的使用方法为

```java
Request request = new Request.Builder()
            .url("http://blog.csdn.net/briblue")
            .cacheControl(Cache.FORCE_NETWORK)
            .build();
```

还有一种情况将 `maxAge` 设置为 0，也不会取缓存，直接走网络。

```java
Request request = new Request.Builder()
            .url("http://blog.csdn.net/briblue")
            .cacheControl(new CacheControl.Builder()
            .maxAge(0, TimeUnit.SECONDS))
            .build();
```

# 总结

- http 协议下 Cache-Control 等消息头的作用
- okhttp 如何用拦截器添加 Cache-Control 消息头进行缓存定制
- okhttp 如何用 CacheControl 进行缓存的控制



[Android 网络请求心路历程](http://www.jianshu.com/p/3141d4e46240#HTTP%E7%BC%93%E5%AD%98%E6%9C%BA%E5%88%B6)

[OkHttp3 源码分析[缓存策略]](http://www.jianshu.com/p/9cebbbd0eeab)

[剖析OkHttp缓存机制](https://github.com/hehonghui/android-tech-frontier/blob/master/issue-42/%E5%89%96%E6%9E%90okhttp%E7%BC%93%E5%AD%98%E6%9C%BA%E5%88%B6.md)

[OKHttp使用Interceptor的缓存问题](http://www.jianshu.com/p/cf59500990c7?utm_source=tuicool&utm_medium=referral)

[使用Retrofit2.0+OkHttp3.0实现缓存处理](https://werb.github.io/2016/07/29/%E4%BD%BF%E7%94%A8Retrofit2+OkHttp3%E5%AE%9E%E7%8E%B0%E7%BC%93%E5%AD%98%E5%A4%84%E7%90%86/)