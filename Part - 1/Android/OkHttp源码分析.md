### 第一步: 创建 OkHttpClient 对象，new OkHttpClient(Builder)

```
OkHttpClient client = new OkHttpClient();
```

这里创建了一个默认的OkHttpCient.Builder，用于配置各种参数。

### 第二步:发起 HTTP 请求，okhttpclient.newCall(request)

```java
String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  Response response = client.newCall(request).execute();
  return response.body().string();
}
```

`OkHttpClient` 实现了 `Call.Factory`，负责根据请求创建新的 `Call`

> `callFactory` 负责创建 HTTP 请求，HTTP 请求被抽象为了 `okhttp3.Call` 类，它表示一个已经准备好，可以随时执行的 HTTP 请求

那我们现在就来看看它是如何创建 Call 的：

```java
/**
  * Prepares the {@code request} to be executed at some point in the future.
  */
@Override public Call newCall(Request request) {
  return new RealCall(this, request);
}
```

如此看来功劳全在 `RealCall` 类了，这里用request对象创建了一个RealCall对象，把一些参数传到RealCall。

### 第三步:execute() or enqueue()

#### 同步网络请求 ：

同步请求，很直接就调用到了最核心的函数数 `getResponseWithInterceptorChain()`。

```java
@Override public Response execute() throws IOException {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");  // (1)
    executed = true;
  }
  try {
    client.dispatcher().executed(this);                                 // (2)
    // 核心的函数
    Response result = getResponseWithInterceptorChain();                // (3)
    if (result == null) throw new IOException("Canceled");
    return result;
  } finally {
    client.dispatcher().finished(this);                                 // (4)
  }
}
```

这里我们做了 4 件事：

1. 检查这个 call 是否已经被执行了，每个 call 只能被执行一次，如果想要一个完全一样的 call，可以利用 `call#clone` 方法进行克隆。
2. 利用 `client.dispatcher().executed(this)` 来进行实际执行，`dispatcher` 是刚才看到的 `OkHttpClient.Builder` 的成员之一，它的文档说自己是异步 HTTP 请求的执行策略，现在看来，同步请求它也有掺和。
3. 调用 `getResponseWithInterceptorChain()` 函数获取 HTTP 返回结果，从函数名可以看出，这一步还会进行一系列“拦截”操作。
4. 最后还要通知 `dispatcher` 自己已经执行完毕。

dispatcher 这里我们不过度关注，在同步执行的流程中，涉及到 dispatcher 的内容只不过是告知它我们的执行状态，比如开始执行了（调用 `executed`），比如执行完毕了（调用 `finished`），在异步执行流程中它会有更多的参与。

真正发出网络请求，解析返回结果的，还是 `getResponseWithInterceptorChain`参见第四步。

#### 发起异步网络请求

```java
client.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        System.out.println(response.body().string());
    }
});

// RealCall#enqueue
@Override public void enqueue(Callback responseCallback) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

将用户接口的`responseCallback`对象封装成一个`AsyncCall`对象提交给`Dispather`来处理，这里的`AsyncCall`是`RealCall`的一个内部类。再看下这个`Dispather`怎么处理这个`AsyncCall`的。

```java
// Dispatcher#enqueue
synchronized void enqueue(AsyncCall call) {
  if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
    runningAsyncCalls.add(call);
    executorService().execute(call);
  } else {
    readyAsyncCalls.add(call);
  }
}
```

这里 dispatcher 管理了一些请求队列，如果正在执行的异步请求没有达到上限，那就直接将这个请求提交给线程池，否则加入 `readyAsyncCalls` 队列，而正在执行的请求执行完毕之后，会调用 `promoteCalls()` 函数，来把 `readyAsyncCalls` 队列中的 `AsyncCall` “提升”为 `runningAsyncCalls`，并开始执行。

```java
//RealCall.java
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }
    ......
    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```

这里的 `AsyncCall` 是 `RealCall` 的一个内部类，它实现了 `Runnable`，所以可以被提交到 `ExecutorService` 上执行，而它在执行时会调用 `getResponseWithInterceptorChain()` 函数，并把结果通过 `responseCallback` 传递给上层使用者。而且在`execute()`里会回调用户接口`responseCallback`的回调方法。

**注意**:这里的回调是在非主线程直接回调的，也就是在Android里使用的话要注意这里面不能直接更新UI操作。

**这样看来，同步请求和异步请求的原理是一样的，都是在 `getResponseWithInterceptorChain()` 函数中通过 `Interceptor` 链条来实现的网络请求逻辑，而异步则是通过 `ExecutorService` 实现。**

### 第四步:getResponseWithInterceptorChain()

```java
//RealCall.java
private Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  interceptors.addAll(client.interceptors());
  interceptors.add(retryAndFollowUpInterceptor);
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  interceptors.add(new CacheInterceptor(client.internalCache()));
  interceptors.add(new ConnectInterceptor(client));
  if (!retryAndFollowUpInterceptor.isForWebSocket()) {
    interceptors.addAll(client.networkInterceptors());
  }
  interceptors.add(new CallServerInterceptor(
      retryAndFollowUpInterceptor.isForWebSocket()));

  Interceptor.Chain chain = new RealInterceptorChain(
      interceptors, null, null, null, 0, originalRequest);
  return chain.proceed(originalRequest);
}
```

在这个方法里就是添加了一些拦截器，然后启动一个拦截器调用链，拦截器递归调用之后最后返回请求的响应`Response`。这里的拦截器分层的思想就是借鉴的网络里的分层模型的思想。请求从最上面一层到最下一层，响应从最下一层到最上一层，每一层只负责自己的任务，对请求或响应做自己负责的那块的修改。

Q1:这里为什么每次都重新创建`RealInterceptorChain`对象，为什么不直接复用上一层的`RealInterceptorChain`对象？(文末给出答案)

A：每次重新创建一个`RealInterceptorChain`对象，因为这里是递归调用，在调用下一层拦截器的interupter()方法的时候，本层的 response阶段还没有执行完成，如果复用`RealInterceptorChain`对象，必然导致下一层修改`RealInterceptorChain`，所以需要重新创建`RealInterceptorChain`对象。

### OkHttp拦截器分层结构

从 `getResponseWithInterceptorChain` 函数我们可以看到，`Interceptor.Chain` 的分布依次是：

[![img](https://ww4.sinaimg.cn/large/006tNbRwly1fdb4w7y0h0j30o90jignd.jpg)](https://ww4.sinaimg.cn/large/006tNbRwly1fdb4w7y0h0j30o90jignd.jpg)



1. 在配置 `OkHttpClient` 时设置的 `interceptors`；
2. 负责失败重试以及重定向的 `RetryAndFollowUpInterceptor`；
3. 负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的响应转换为用户友好的响应的 `BridgeInterceptor`；
4. 负责读取缓存直接返回、更新缓存的 `CacheInterceptor`；
5. 负责和服务器建立连接的 `ConnectInterceptor`；
6. 配置 `OkHttpClient` 时设置的 `networkInterceptors`；
7. 负责向服务器发送请求数据、从服务器读取响应数据的 `CallServerInterceptor`。

在这里，位置决定了功能，最后一个 Interceptor 一定是负责和服务器实际通讯的，重定向、缓存等一定是在实际通讯之前的。

[责任链模式](https://zh.wikipedia.org/wiki/%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F)在这个 `Interceptor` 链条中得到了很好的实践。这样的好处是将请求的发送和处理分开，并且可以动态添加中间的处理方实现对请求的处理、短路等操作。

> 它包含了一些命令对象和一系列的处理对象，每一个处理对象决定它能处理哪些命令对象，它也知道如何将它不能处理的命令对象传递给该链中的下一个处理对象。该模式还描述了往该处理链的末尾添加新的处理对象的方法。

对于把 `Request` 变成 `Response` 这件事来说，每个 `Interceptor` 都可能完成这件事，所以我们循着链条让每个 `Interceptor` 自行决定能否完成任务以及怎么完成任务（自力更生或者交给下一个 `Interceptor`）。这样一来，完成网络请求这件事就彻底从 `RealCall` 类中剥离了出来，简化了各自的责任和逻辑。两个字：优雅！

责任链模式在安卓系统中也有比较典型的实践，例如 view 系统对点击事件（TouchEvent）的处理，具体可以参考[Android设计模式源码解析之责任链模式](https://github.com/simple-android-framework-exchange/android_design_patterns_analysis/tree/master/chain-of-responsibility/AigeStudio#android源码中的模式实现)中相关的分析。



```java
//RealInterceptorChain.java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
  Connection connection) throws IOException {
if (index >= interceptors.size()) throw new AssertionError();
calls++;
......
RealInterceptorChain next = new RealInterceptorChain(
    interceptors, streamAllocation, httpCodec, connection, index + 1, request);
Interceptor interceptor = interceptors.get(index);
Response response = interceptor.intercept(next);
...
return response;
}
```

RealInterceptorChain的`proceed()`，每次重新创建一个RealInterceptorChain对象，然后调用下一层的拦截器的`interceptor.intercept()`方法。
每一个拦截器的`intercept()`方法都是这样的模型

```java
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    // 1、该拦截器在Request阶段负责的事情

    // 2、调用RealInterceptorChain.proceed()，其实是递归调用下一层拦截器的intercept方法
    response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);

    //3、该拦截器在Response阶段负责的事情，然后返回到上一层拦截器的 response阶段
    return  response;     
    }
  }
```

这差不多就是OkHttp的分层拦截器模型，借鉴了网络里的OSI七层模型的思想。最底层是`CallServerInterceptor`，类比网络里的`物理层`。OkHttp还支持用户自定义拦截器，插入到最顶层和`CallServerInterceptor`上一层的位置。比如官方写了一个`Logging Interceptor`，用于打印网络请求日志的拦截器。

#### BridgeInterceptor

```java
Request userRequest = chain.request();
Request.Builder requestBuilder = userRequest.newBuilder();
// Request阶段
RequestBody body = userRequest.body();
if (body != null) {
  MediaType contentType = body.contentType();
    ......
  long contentLength = body.contentLength();
  if (contentLength != -1) {
    requestBuilder.header("Content-Length", Long.toString(contentLength));
    requestBuilder.removeHeader("Transfer-Encoding");
  } else {
    requestBuilder.header("Transfer-Encoding", "chunked");
    requestBuilder.removeHeader("Content-Length");
  }
  if (userRequest.header("Connection") == null) {
  requestBuilder.header("Connection", "Keep-Alive");
 }
}
    .....
Response networkResponse = chain.proceed(requestBuilder.build());
// Response阶段
    .....
if (transparentGzip
    && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
    && HttpHeaders.hasBody(networkResponse)) {
  GzipSource responseBody = new GzipSource(networkResponse.body().source());
  Headers strippedHeaders = networkResponse.headers().newBuilder()
      .removeAll("Content-Encoding")
      .removeAll("Content-Length")
      .build();
  responseBuilder.headers(strippedHeaders);
  responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
}
```

BridgeInterceptor拦截器在Request阶段，将用户的配置信息，重新创建Request.Builder对象，重新build出Request对象，并添加一些请求头，比如:host，content-length，keep-alive等。
BridgeInterceptor在Response阶段做gzip解压操作。

#### CacheInterceptor

CacheInterceptor拦截器在Request阶段判断该请求是否有缓存，是否需要重新请求，如果不需要重新请求，直接从缓存里取出内容，封装一个Response返回，不需要再调用下一层。
CacheInterceptor拦截器在Response阶段，就是把下面一层的Response做缓存。

#### 建立连接：ConnectInterceptor

```java
@Override public Response intercept(Chain chain) throws IOException {
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Request request = realChain.request();
  StreamAllocation streamAllocation = realChain.streamAllocation();

  // We need the network to satisfy this request. Possibly for validating a conditional GET.
  boolean doExtensiveHealthChecks = !request.method().equals("GET");
  HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
  RealConnection connection = streamAllocation.connection();

  return realChain.proceed(request, streamAllocation, httpCodec, connection);
}
```

ConnectInterceptor拦截器只在Request阶段建立连接，Response阶段直接把下一层的Response返回给上一层。再看下建立连接的过程。

```java
public HttpCodec newStream(OkHttpClient client, boolean doExtensiveHealthChecks) {
....
try {
  RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
      writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);
  HttpCodec resultCodec = resultConnection.newCodec(client, this);
......
} catch (IOException e) {
  throw new RouteException(e);
}
}
```

实际上建立连接就是创建了一个 `HttpCodec` 对象，概括来说，就是找到一个可用的 `RealConnection`，再利用 `RealConnection` 的输入输出（`BufferedSource` 和 `BufferedSink`）创建 `HttpCodec` 对象，供后续步骤使用。



**findHealthyConnection()**函数寻找一条健康的网络连接，其内部主要调用了`findConnection()`。

```java
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
  boolean connectionRetryEnabled) throws IOException {
Route selectedRoute;
synchronized (connectionPool) {
 .....
  // Attempt to get a connection from the pool.
  Internal.instance.get(connectionPool, address, this);
  if (connection != null) {
    return connection;
  }

  selectedRoute = route;
}

// If we need a route, make one. This is a blocking operation.
if (selectedRoute == null) {
  selectedRoute = routeSelector.next();
}

// Create a connection and assign it to this allocation immediately. This makes it possible for
// an asynchronous cancel() to interrupt the handshake we're about to do.
RealConnection result;
synchronized (connectionPool) {
  route = selectedRoute;
  refusedStreamCount = 0;
  result = new RealConnection(connectionPool, selectedRoute);
  acquire(result);
  if (canceled) throw new IOException("Canceled");
}

// Do TCP + TLS handshakes. This is a blocking operation.
result.connect(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);
routeDatabase().connected(result.route());

Socket socket = null;
synchronized (connectionPool) {
  // Pool the connection.
  Internal.instance.put(connectionPool, result);
 .....
}
closeQuietly(socket);
return result;
}
```

**这里面大概就是从连接池里去找已有的网络连接，如果有，则复用，减少三次握手；没有的话，则创建一个RealConnection对象，三次握手，建立连接，然后将连接放到连接池。具体的内部`connect`过程，就不深入了。**

```
public ConnectionPool() {
this(5, 5, TimeUnit.MINUTES);
}
public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
}
```

`ConnectionPool`最多支持保持5个地址的连接keep-alive，每个keep-alive 5分钟，并有异步线程循环清理无效的连接。

#### 发送和接收数据：CallServerInterceptor

```java
@Override public Response intercept(Chain chain) throws IOException {
  HttpCodec httpCodec = ((RealInterceptorChain) chain).httpStream();
  StreamAllocation streamAllocation = ((RealInterceptorChain) chain).streamAllocation();
  Request request = chain.request();

  long sentRequestMillis = System.currentTimeMillis();
  httpCodec.writeRequestHeaders(request);

  if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
    Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
    BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
    request.body().writeTo(bufferedRequestBody);
    bufferedRequestBody.close();
  }

  httpCodec.finishRequest();

  Response response = httpCodec.readResponseHeaders()
      .request(request)
      .handshake(streamAllocation.connection().handshake())
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build();

  if (!forWebSocket || response.code() != 101) {
    response = response.newBuilder()
        .body(httpCodec.openResponseBody(response))
        .build();
  }

  if ("close".equalsIgnoreCase(response.request().header("Connection"))
      || "close".equalsIgnoreCase(response.header("Connection"))) {
    streamAllocation.noNewStreams();
  }

  // 省略部分检查代码

  return response;
}
```

CallServerInterceptor 精简出来的代码就是 writeRequestHeaders()，flushRequest()，finishRequest()，发送请求，然后readResponseHeaders，openResponseBody 读取 response。

我们抓住主干部分：

1. 向服务器发送 request header；
2. 如果有 request body，就向服务器发送；
3. 读取 response header，先构造一个 `Response` 对象；
4. 如果有 response body，就在 3 的基础上加上 body 构造一个新的 `Response` 对象；

这里我们可以看到，核心工作都由 `HttpCodec` 对象完成，而 `HttpCodec` 实际上利用的是 Okio，而 Okio 实际上还是用的 `Socket`，所以没什么神秘的，只不过一层套一层，层数有点多。

其实 `Interceptor` 的设计也是一种分层的思想，每个 `Interceptor` 就是一层。为什么要套这么多层呢？分层的思想在 TCP/IP 协议中就体现得淋漓尽致，分层简化了每一层的逻辑，每层只需要关注自己的责任（单一原则思想也在此体现），而各层之间通过约定的接口/协议进行合作（面向接口编程思想），共同完成复杂的任务。



`Interceptor` 是 OkHttp 最核心的一个东西，不要误以为它只负责拦截请求进行一些额外的处理（例如 cookie），**实际上它把实际的网络请求、缓存、透明压缩等功能都统一了起来**，每一个功能都只是一个 `Interceptor`，它们再连接成一个 `Interceptor.Chain`，环环相扣，最终圆满完成一次网络请求。最后补充几个关于OkHttp的面试问题。

- OkHttp是如何做链路复用？
- OkHttp的Intereptor能不能取消一个request？ **不能**
  这两个问题在分析源码之后应该很容易回答了。

**注意：当我们在自己实现Interceptor时并复写intercept方法时，一定要记住调用chain.proceed方法，否则上面提到的循环递归则会终止，也就是说最终真正的发送网络请求并获取结果的getResponse方法就不会被调用**



### 核心重点类Dispatcher线程池介绍

```java
public final class Dispatcher {
  /** 最大并发请求数为64 */
  private int maxRequests = 64;
  /** 每个主机最大请求数为5 */
  private int maxRequestsPerHost = 5;

  /** 线程池 */
  private ExecutorService executorService;

  /** 准备执行的请求 */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** 正在执行的异步请求，包含已经取消但未执行完的请求 */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** 正在执行的同步请求，包含已经取消单未执行完的请求 */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

在OkHttp，使用如下构造了单例线程池

```java
public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```

构造一个线程池ExecutorService：

```java
executorService = new ThreadPoolExecutor(
//corePoolSize 最小并发线程数,如果是0的话，空闲一段时间后所有线程将全部被销毁
    0, 
//maximumPoolSize: 最大线程数，当任务进来时可以扩充的线程最大值，当大于了这个值就会根据丢弃处理机制来处理
    Integer.MAX_VALUE, 
//keepAliveTime: 当线程数大于corePoolSize时，多余的空闲线程的最大存活时间
    60, 
//单位秒
    TimeUnit.SECONDS,
//工作队列,先进先出
    new SynchronousQueue<Runnable>(),   
//单个线程的工厂         
   Util.threadFactory("OkHttp Dispatcher", false));
```

可以看出，在Okhttp中，构建了一个核心为[0, Integer.MAX_VALUE]的线程池，它不保留任何最小线程数，随时创建更多的线程数，当线程空闲时只能活60秒，它使用了一个不存储元素的阻塞工作队列，一个叫做"OkHttp Dispatcher"的线程工厂。

也就是说，在实际运行中，当收到10个并发请求时，线程池会创建十个线程，当工作完成后，线程池会在60s后相继关闭所有线程。

```java
synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
}
```

从上述源码分析，如果当前还能执行一个并发请求，则加入 runningAsyncCalls ，立即执行，否则加入 readyAsyncCalls 队列。

#### Dispatcher线程池总结

1）调度线程池Disptcher实现了高并发，低阻塞的实现
2）采用Deque作为缓存，先进先出的顺序执行
3）任务在try/finally中调用了finished函数，控制任务队列的执行顺序，而不是采用锁，减少了编码复杂性提高性能

```java
 try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } finally {
        client.dispatcher().finished(this);
      }
```

当任务执行完成后，无论是否有异常，finally代码段总会被执行，也就是会调用Dispatcher的finished函数

```java
void finished(AsyncCall call) {
  finished(runningAsyncCalls, call, true);
}
```

从上面的代码可以看出，第一个参数传入的是正在运行的异步队列，第三个参数为true，下面再看有是三个参数的finished方法：

```java
private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }
```

打开源码，发现它将正在运行的任务Call从队列runningAsyncCalls中移除后，获取运行数量判断是否进入了Idle状态,接着执行promoteCalls()函数,下面是promoteCalls()方法：

```java
private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();

      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }

      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }
```

主要就是遍历等待队列，并且需要满足同一主机的请求小于maxRequestsPerHost时，就移到运行队列中并交给线程池运行。就主动的把缓存队列向前走了一步，而没有使用互斥锁等复杂编码



### 返回数据的获取

在同步（`Call#execute()` 执行之后）或者异步（`Callback#onResponse()` 回调中）请求完成之后，我们就可以从 `Response` 对象中获取到响应数据了，包括 HTTP status code，status message，response header，response body 等。这里 body 部分最为特殊，因为服务器返回的数据可能非常大，所以必须通过数据流的方式来进行访问（当然也提供了诸如 `string()` 和 `bytes()` 这样的方法将流内的数据一次性读取完毕），而响应中其他部分则可以随意获取。

响应 body 被封装到 `ResponseBody` 类中，该类主要有两点需要注意：

1. 每个 body 只能被消费一次，多次消费会抛出异常；
2. body 必须被关闭，否则会发生资源泄漏；

在发送和接收数据：CallServerInterceptor 中，我们就看过了 body 相关的代码：

```
if (!forWebSocket || response.code() != 101) {
  response = response.newBuilder()
      .body(httpCodec.openResponseBody(response))
      .build();
}
```

由 `HttpCodec#openResponseBody` 提供具体 HTTP 协议版本的响应 body，而 `HttpCodec` 则是利用 Okio 实现具体的数据 IO 操作。

这里有一点值得一提，OkHttp 对响应的校验非常严格，HTTP status line 不能有任何杂乱的数据，否则就会抛出异常，在我们公司项目的实践中，由于服务器的问题，偶尔 status line 会有额外数据，而服务端的问题也毫无头绪，导致我们不得不忍痛继续使用 HttpUrlConnection，而后者在一些系统上又存在各种其他的问题，例如魅族系统发送 multi-part form 的时候就会出现没有响应的问题。

### HTTP 缓存

在同步网络请求 小节中，我们已经看到了 `Interceptor` 的布局，在建立连接、和服务器通讯之前，就是 `CacheInterceptor`，在建立连接之前，我们检查响应是否已经被缓存、缓存是否可用，如果是则直接返回缓存的数据，否则就进行后面的流程，并在返回之前，把网络的数据写入缓存。

这块代码比较多，但也很直观，主要涉及 HTTP 协议缓存细节的实现，而具体的缓存逻辑 OkHttp 内置封装了一个 `Cache` 类，它利用 `DiskLruCache`，用磁盘上的有限大小空间进行缓存，按照 LRU 算法进行缓存淘汰，这里也不再展开。

我们可以在构造 `OkHttpClient` 时设置 `Cache` 对象，在其构造函数中我们可以指定目录和缓存大小：

```
public Cache(File directory, long maxSize);
```

而如果我们对 OkHttp 内置的 `Cache` 类不满意，我们可以自行实现 `InternalCache` 接口，在构造 `OkHttpClient` 时进行设置，这样就可以使用我们自定义的缓存策略了。



### 总结

OkHttp的底层是通过Java的Socket发送HTTP请求与接受响应的(这也好理解，HTTP就是基于TCP协议的)，但是OkHttp实现了连接池的概念，即对于同一主机的多个请求，其实可以公用一个Socket连接，而不是每次发送完HTTP请求就关闭底层的Socket，这样就实现了连接池的概念。而OkHttp对Socket的读写操作使用的OkIo库进行了一层封装。

在文章最后我们再来回顾一下完整的流程图：

![okhttp_full_process](https://blog.piasy.com/img/201607/okhttp_full_process.png)

- `OkHttpClient` 实现 `Call.Factory`，负责为 `Request` 创建 `Call`；
- `RealCall` 为具体的 `Call` 实现，其 `enqueue()` 异步接口通过 `Dispatcher` 利用 `ExecutorService` 实现，而最终进行网络请求时和同步 `execute()` 接口一致，都是通过 `getResponseWithInterceptorChain()` 函数实现；
- `getResponseWithInterceptorChain()` 中利用 `Interceptor` 链条，分层实现缓存、透明压缩、网络 IO 等功能；

### 问题

1. 如何使用OkHttp进行异步网络请求，并根据请求结果刷新UI
2. 可否介绍一下OkHttp的整个异步请求流程
3. OkHttp对于网络请求都有哪些优化，如何实现的
4. OkHttp框架中都用到了哪些设计模式


![](https://ww2.sinaimg.cn/large/006tNbRwly1fdb4sqjt8rj30ct0mlmy7.jpg)