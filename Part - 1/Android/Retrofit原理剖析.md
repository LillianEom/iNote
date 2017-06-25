## Retrofit原理剖析



## 网络请求框架的基本流程

1. 构建Request，入队
2. 进入 Executor 执行，Looper 不断循环，拿到 Request 执行
3. 执行结果的解析与返回

### 构建 Request

Retrofit 通过注解来构建 Request，所有的参数都可以简单配置完毕，上层不需要关心底层使用何种方式实现构建 Request 过程。

```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

### 实现 Executor

实现上述声明的接口，得到可以做具体请求操作的实现类。

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

**相关概念**
**CallAdapter：**Adapter a Call into the type of T.（将一个Call适配为另外一个Call的适配器接口）

- Type responseType()：将返回的请求转化为参数Type类型
- T adapt(Call<R> call)：将一个Call转换成另外一个Call

### HTTP Call

调用上述实现类的方法即可得到 Call，而 Retrofit 内部使用 OkHttp 实现 HTTP Call，最后通过 Converter 转换成用户想要的对象体。

Call：An invocation of a Retrofit method that sends a request to a webserver and returns a response.（请求发送与响应返回方法的调用）

```java
Call<List<Repo>> repos = service.listRepos("octocat");
```

### 主线用例

```java
// 源码剖析 - 步骤1
Retrofit retrofit = new Retrofit.Builder()
            .baseUrl("http://gank.io/")
            .addConverterFactory(GsonConverterFactory.create())
            .build();
// 源码剖析 - 步骤2
Api api = retrofit.create(Api.class);
// 源码剖析 - 步骤4
Call<BaseModel<ArrayList<Benefit>>> call = api.defaultBenefits(20, page++);

call.enqueue(new Callback<BaseModel<ArrayList<Benefit>>>() {
		    @Override
		    public void onResponse(Call<BaseModel<ArrayList<Benefit>>> call, Response<BaseModel<ArrayList<Benefit>>> response) {
		        if (action == PullRecycler.ACTION_PULL_TO_REFRESH) {
		            mDataList.clear();
		        }
		        if (response.body().results == null || response.body().results.size() == 0) {
		            recycler.enableLoadMore(false);
		        } else {
		            recycler.enableLoadMore(true);
		            mDataList.addAll(response.body().results);
		            adapter.notifyDataSetChanged();
		        }
		        recycler.onRefreshCompleted();
		    }

		    @Override
		    public void onFailure(Call<BaseModel<ArrayList<Benefit>>> call, Throwable t) {
		        recycler.onRefreshCompleted();
		    }
		});
```

## Retrofit主线源码剖析

Retrofit 精髓就在三点

- 1、动态代理，用注解来生成请求参数；
- 2、适配器模式的应用，请求返回各种 CallAdapter，可扩展到 RxJava、Java8，还有任何你自己写的 Adapter；
- 3、Converter，你可以把请求的响应用各种 Converter 转成你的需求上。

上面过程可以看成四步，也是接下来要分析的四个步骤。
1、Retrofit.Builder().build()
2、retrofit.create
3、github.contributors
4、call.enqueue



### 第一步、Retrofit.Builder().build()

**构建Retrofit对象主要思路(就是用到建造者模式)：**

1. 创建 callbackExecutor（内部获取主线程 MainLooper 来构建 Hanlder，其 execute 方法本质是 Handler.post(runnable)，待用于线程切换）；
2. 构建 adapterFactories 集合，将 defaultAdapterFactory 加入其中（ExecutorCallAdapterFactory 类，线程切换关键实现，内部持有 OkHttp 代理 delegate，在 delegate.enqueue 中的 onRespond 方法内使用刚刚创建的 callbackExecutor.execute 方法，从而实现线程切换）

```java
public Retrofit build() {
	if (baseUrl == null) {
		throw new IllegalStateException("Base URL required.");
	}
	// 忽略不看 这个工厂模式基本没什么作用了，目前来说Retrofit只支持OkHttp，不能支持URLConnection或者HTTPClient
	okhttp3.Call.Factory callFactory = this.callFactory;
	if (callFactory == null) {
		//这里首先创建了一个OkHttpClient对象
		callFactory = new OkHttpClient();
	}

	//然后创建Executor对象
	Executor callbackExecutor = this.callbackExecutor;
	if (callbackExecutor == null) {
		callbackExecutor = platform.defaultCallbackExecutor();
	}

	//然后创建 CallAdapter对象,
	// 构建adapterFactories集合，将defaultAdapterFactory加入其中
	// Make a defensive copy of the adapters and add the default Call adapter.
	List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
	adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
	// 创建Converter对象
	// 一般传入的为GsonConverterFactory对象，其作用主要是将json转换成java对象
	// Make a defensive copy of the converters.
	List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);
	//最终完成Retrofit对象构建
	return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
	  callbackExecutor, validateEagerly);
}
```

总结这步的步骤就是准备好要用到的东西，callAdapter 是请求返回的对象，Converter 是转换器，转换请求的响应到对应的实体对象，OkHttpClient 是具体的 OkHttp 的请求客户端。

### 第二步、Retrofit.create()

```java
// create方法内部实现，返回一个动态代理实例  
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
// 该动态代理会对接口方法进行拦截
return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
    // 创建一个InvocationHandler，接口方法被调用时会被拦截调用                            
    new InvocationHandler() {
      private final Platform platform = Platform.get();
	// 当Api接口方法被调用时，会调用invoke方法拦截
      @Override public Object invoke(Object proxy, Method method, Object... args)
          throws Throwable {
        // If the method is a method from Object then defer to normal invocation.
        if (method.getDeclaringClass() == Object.class) {
          return method.invoke(this, args);
        }
        if (platform.isDefaultMethod(method)) {
          return platform.invokeDefaultMethod(method, service, proxy, args);
        }
        // 通过解析api方法注解、传参，将接口中方法适配成HTTP Call
        ServiceMethod serviceMethod = loadServiceMethod(method);
        // 拦截下来之后，内部构建一个okHttpCall
        OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
        // 最后通过callAdapter将okHttpCall转换成为Retrofit适用的call delegates（代理），Android平台默认使用
        //ExecutorCallAdapterFactory，adapt返回ExecutorCallbackCall
        return serviceMethod.callAdapter.adapt(okHttpCall);
      }
    });
}
```

这里就是动态代理的原理，在使用 Retrofit 的时候，我们会把网络请求的接口写成这样一个接口方法。第二步只是准备这样一个代理。

这个方法里，做了三件事情：

1. 解析方法的注解，比如`get/post、url、Headers`等，解析的结果存放于`ServiceMethod`对象中。
2. 创建`OkHttpCall`，该对象负责与 OKHttp3 对接，处理同步或异步的网络请求。
3. 变化无穷的适配

### 第三步、调用这个接口方法

调用的时候才会真正的触发这样一个过程。

```java
public interface GitHub {
    @GET("/repos/{owner}/{repo}/contributors")
    Call<List<Contributor>> contributors(
        @Path("owner") String owner,
        @Path("repo") String repo);
}
```

动态代理就是拦截 GitHub 接口方法调用，但接下来主要做两个工作：1、拿到这个方法的各种注解，然后配置 OkHttp 的网络请求参数。2、把真实的网络请求的 OkHttpCall 转换成该接口对应的 CallAdapter。

#### 3.1、loadServiceMethod()

**ServiceMethod 整体思路：** 内部主要是将方法中的注解取出，转换成 HTTP Call 的逻辑

```java
// 内部有缓存以提高性能，避免重复解析 - 使用 LinkedHashMap 来存储 ServiceMethod，与接口中方法一一对应
private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();
ServiceMethod loadServiceMethod(Method method) {
 ServiceMethod result;
 synchronized (serviceMethodCache) {
   // 通过传入的方法，从 Map 中取出对应的 ServiceMethod
   result = serviceMethodCache.get(method);
   if (result == null) {
 	 // 如果还没有实现，则构造一个 ServiceMethod 实例放入 Map 中
     result = new ServiceMethod.Builder(this, method).build();
     serviceMethodCache.put(method, result);
   }
 }
 return result;
}
```

这里面对每个接口方法的注解解析生成 ServiceMethod 对象，使用了一个缓存，避免一个网络请求在多次调用的时候频繁的解析注解，毕竟注解解析过程消耗比较大。

这里面主要的工作又在 ServiceMethod.Builder().build()；

这里简单来说就是做三件事，1、找到对应的 CallAdapter，2、找到对应的 Converter，3、解析注解生成参数

##### 3.1.1 找到对应的 CallAdapter

```java
for (int i = start, count = adapterFactories.size(); i < count; i++) {
  	CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
  	if (adapter != null) {
    	return adapter;
  	}
}
```

在找对应的 callAdapter 的时候，是根据不同的 CallAdapterFactory 的返回类型来区分的，比如我们看下默认的 ExecutorCallAdapterFactory

```java
public CallAdapter<Call<?>> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public <R> Call<R> adapt(Call<R> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
```

这里就是说返回类型是 Call.class，都是使用 ExecutorCallAdapterFactory 创建的的 ExecutorCallbackCall。而 RxJava 对应的返回类型就是 Observable.class。

##### 3.1.2 找对应的 Converter

```java
  public <T> Converter<ResponseBody, T> nextResponseBodyConverter(Converter.Factory skipPast,
      Type type, Annotation[] annotations) {

    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }
 }
```

看下 GsonConverterFactory

```java
@Override
public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
  Retrofit retrofit) {
	TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
	return new GsonResponseBodyConverter<>(gson, adapter);
}
```

##### 3.1.3 parseMethodAnnotation

最后就是解析请求参数的工作，这部分就是解析注解上配置的网络请求参数。

三步完成之后，ServiceMethod 对象就 build 出来了，然后放到缓存中，避免下次再 build。再回到主线的第三步，loadServiceMethod 完成之后，就是创建一个 OkHttpCall。

#### 3.2、创建 OKhttpCall 并 adapter 对应的call

这个 OkHttpCall 才是真正 OkHttp 请求的回调，但是针对我们使用的不同的回调，比如：RxJava 的 Observable、Call，所以有一层转换的关系，把 OkHttpCall 转成对应的 Observable 和 Call。就是 serviceMethod.callAdapter.adapt( okHttpCall ) 的工作。

就拿默认的 CallAdapter 说吧，这里是个匿名类。

```java
return new CallAdapter<Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public <R> Call<R> adapt(Call<R> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    }
```

他的实现就是创建的一个 ExecutorCallbackCall 作为 Call 返回了。当然它里面还也持有 OkHttpCall 的引用。目的是为了上层的回调，下面会讲。

好了至此，主线的第三步完成，一切准备就绪了，就等网络请求了。网络请求分为异步、同步的，这里只讲异步的。同步的更简单。

### 第四步、call.enqueue

retrofit.create() 方法创建好实现类后，调用接口方法获得 Call，当接口方法被调用时候，会被拦截并且转换成 HTTP Call，adapte 步骤转换的时候，对应的 call 持有的真正网络请求 OkHttp 的 Call。

**下面我们使用 call 进行 enqueue 操作**：此处的 call，即 ExecutorCallbackCall，它有 execute（同步）、enqueue（异步）两个方法，其中 enqueue 方法，可达到子线程请求，成功后切换回主线程的效果，免去了开启线程、使用 handler 跨线程通信的操作。

```java
Call<BaseModel<ArrayList<Benefit>>> call = api.defaultBenefits(20, page++);
call.enqueue(new Callback<BaseModel<ArrayList<Benefit>>>() {
             // 该处的onResponse方法已经转换到主线程上了，而转换关键在于Retrofit对象构建时，defaultAdapterFactory内部实现
              @Override
              public void onResponse(Call<BaseModel<ArrayList<Benefit>>> call, Response<BaseModel<ArrayList<Benefit>>> response) {
                 ...
              }

              @Override
              public void onFailure(Call<BaseModel<ArrayList<Benefit>>> call, Throwable t) {
                 ...
              }
          }
```



### 第五步、Converter的转换

就在第四步请求的时候，真实的网络请求 OkHttpCall 在 enque 的时候，里面有个 onResponse，这里面调用了一个 parseResponse

```java
 Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
     T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```

这里面关键的又回调了 serviceMethod.toResponse，这里毫无疑问的就是找到把之前找出的 Converter 拿出来，进行 convert 操作。

```java
T toResponse(ResponseBody body) throws IOException {
   return responseConverter.convert(body);
}
```

至此一个完整的 Retrofit 的网络请求就已经完成了！



## Retrofit 主线流程总结：

1. **Retrofit对象的构建 - Retrofit.Builder()...build():**

   ①**构建OkHttpClient**，目前Retrofit仅支持OkHttpClient；

   ②**构建Executor**：优先根据用户提供的callBackExcecutor来构建，若用户没有提供，则提供defaultCallbackExecutor（其**内部会获取MainLooper构建handler，execute方法直接handler.post(runnable)，实现在主线程上的操作**）；

   ③**使用executor来构建adapterFactories集合**，优先将用户提供的adapterFactory加入到其中，再加上defaultCallAdapterFactory（传入②创建的callbackExecutor，defaultCallAdapterFactory内部持有OkHttpCall，在其enqueue方法中的onResponse方法调用defaultCallbackExecutor.execute方法，从而**实现线程切换操作**）；

   ④最终使用**Retrofit构造方法**构建Retrofit实例对象

2. **Retrofit接口方法的实现方式 - retrofit.create(接口名.class)：**

   ①**create方法创建并返回动态代理对象实例**，动态代理对象内部会拦截接口方法的调用

   ②**动态代理内部通过ServiceMethod将接口方法适配成HTTP Call**，再构造对应的OkHttpCall，最后通过CallAdapter转换成Retrofit适用的call delegate（ExecutorCallbackCall）。

3. **使用动态代理(接口实现类)调用接口方法得到Call、使用call.enqueue进行异步请求：**

   ①调用接口方法时，动态代理对象（接口实现类）**内部拦截**；

   ②**调用call.enqueue**，内部会调用ExecutorCallAdapter的enqueue方法，enqueue中onResponse方法调用defaultCallbackExecutor.execute方法，使用主线程Handler.post(runnable)从而**实现线程切换操作**。

Retrofit主线流程总结图解（从上往下阅读）：



![img](http://upload-images.jianshu.io/upload_images/1513860-c760c0ae59e266c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



**Retrofit工作主要代码**

```java
ServiceMethod serviceMethod = loadServiceMethod(method);
OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
return serviceMethod.callAdapter.adapt(okHttpCall);
```

![](https://blog.piasy.com/img/201606/retrofit_stay.png)





Retrofit火就在于它能够扩张各种CallbackFactory，各种Conveter。当然配合 RxJavaCallbackFactory 也是它火的一个原因。



[《公共技术点之 Java 动态代理》](http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)

[我对Retrofit的认识](http://www.jianshu.com/p/bee3deda41a6)

[拆轮子系列 - 如何由浅入深探索 Retrofit 源码？](http://www.jianshu.com/p/9f617a43d579)

[Retrofit2 完全解析 探索与okhttp之间的关系（二）](https://mp.weixin.qq.com/s/e87Totg1ipVRjCjI4eYOCA)





