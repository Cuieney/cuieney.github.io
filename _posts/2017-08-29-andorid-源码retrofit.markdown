---
layout:     post
title:      "一张图带你走进Retrofit源码世界"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 源码
---

> “Yeah It's on. ”


### 前提
> 只有了解了框架的原理才能更好的使用她，才能定位问题的根本。写这篇文章的也是为了自我的学习和提升。其实看源码就跟看书一样，看了这么多本书有什么用呢，其实不然，这些知识已经潜移默化的影响了你的思维。你之后在阅读源码时，会发现能更快的上手了。
> 
> 引用别人的一句话：当我还是个孩子时吃的很多食物，大部分已经一去不复返而且被我忘掉了，但可以肯定的是，它们中的一部分已经长成我的骨头和肉

#### 友情提醒
1.这篇文章主要讲retrofit如何request 和 response
2.不会详细到每个api
3.文章会以一个flow 来讲解

### 上图
>如果下图有错误欢迎评论指正，如果看不清你可以下载下来放大看，应该会好点。我们这次会以这个图的flow 来讲解（主要是左半边）。

![retrofit_flow.png](http://upload-images.jianshu.io/upload_images/3415839-d247cc5e90ba3ef6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### retrofit初始化配置
>这里讲解的就是上图中的adapterFactories，咱们这边以RxjavaAdapter来讲解，下面是一些retrofit的初始化配置。

```
 private Retrofit createRetrofit(OkHttpClient client, String url) {
        return new Retrofit.Builder()
                .baseUrl(url)
                .client(client)
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .addConverterFactory(GsonConverterFactory.create())
                .build();
    }
```
这里我们只说一下主要的东西，大家都知道retrofit的adapter 我们根据我们api请求的不同设置不同的adapter来让retrofit执行不同的操作，当我们在设置他的CallAdapter时，点进源码可以看到，retrofit把这个CallAdapter存入了一个集合中

```
final List<CallAdapter.Factory> adapterFactories = new ArrayList<>();

```

然后通过build把这个集合回传给了retrofit的这个全局边变量`List<CallAdapter.Factory> adapterFactories`保存着这些CallAdapter，仔细看的同学会发现，retrofit会默认给你添加一个CallAdapter

```
public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      
      //这里就是默认的adapter 防止用户未设置。
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
  }

```
以上就说到这里，知道有个CallAdapterFactory集合存在Retrofit的全局变量中。

### 配置Api
>这里讲解的是图中的这方法`create(final Class<T> service) `稍微提一下吧。上面的adapter其实是根据你在Api中配置的返回参数有关。当我们要使用Rxjava时我们传对应的Rxjava的adapter 如果我们配置的是okhttp的Call回调，自然我们都不需要配置adapter，retrofit都给我们默认配置好了，下图可以看到两种不同的response

```
public interface Api {

    @Multipart
    @POST("upload")
    Observable<BaseBean> postVoice(@Part("deviceNo") RequestBody deviceNo,
                                   @Part("duration") RequestBody duration,
                                   @Part("title") RequestBody title,
                                   @Part MultipartBody.Part file);
    @GET("selfList")
    Call<VoiceList> getVoiceList(@Query("deviceNo")String deviceNo);

}

```
### Request
>这里讲解的是上图中的
>`create(final Class<T> service) ` 和`ServiceMethod`

当我们要发起请求的时候，我们是不是这样做，拿到之前配置好的Retrofit然后调用他的Create方法然后把相应的api传入.
`Retroift.create(Api.class)`这样的操作

现在我们进入源码查看这个retrofit最亮点的地方（create（api.class）），通过反射实例化接口，然后拦截接口中的方法，来做api的请求。返回给这个method。

先把源码晾上然后讲解

```
 public <T> T create(final Class<T> service) {
                ·······省略的代码·······
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
                   ·······省略的代码·······
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }

```
上面的代码已经是把无关紧要的代码剔除了，我们只看重要的代码

* 我们可以看到create这个方法通过代理反射实例化我们传进来的api接口
* 然后通过`ServiceMethod`和`OkHttpCall`来拿到这个方法的相关参数，来invoke这个方法
* 最后通过`serviceMethod.callAdapter.adapt(okHttpCall)`把请求到的值返回回去

***_其实在最后一步大家应该明白了，请求是来自于ServiceMethod的callAdapter参数，然后通过adapt来进行okhttp请求。_***

下面我会讲解这个calladapter 如何而来。


### ServiceMethod如何发起请求

这是一个流程哈，我们这时候进入代码core部分，他的ServiceMethod时通过retrofit的loadServiceMethod拿到的实例化对象。我们进入这个方法可以看到其实这是一个全局的ServiceMethod的cacheMap

```
 ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }

```
但是不论他做不做缓存肯定有new 的地方，然后才能加入缓存中吧。从上图代码中我们可以看到这个new 的地方。

```
if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }

```
这下清楚了ServiceMethod哪里来的了吧，通过`new ServiceMethod.Builder<>(this, method).build()`这句话来的。看源码不都这样吗！一步一步的进入源码然后理解他的每一层 意思和目的。to be continue

上ServiceMethod.Builder的源码，我们只看重要的代码，其他的忽略

```
 Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }

    public ServiceMethod build() {
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();
      ·······省略的代码·······
      }

```
小伙子们，是不是看到了我们一直最关心的calladapter了，在build的时候通过createCallAdapter来初始化了这个值。进入方法看个究竟

```
private CallAdapter<T, R> createCallAdapter() {
      Type returnType = method.getGenericReturnType();
      ·······省略的代码·······
      Annotation[] annotations = method.getAnnotations();
      try {
        ····敲黑板，敲黑板，敲黑板·····
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
    }

```
看到了吧，小伙伴这个calladapter 来自于retrofit里面的方法，我们在loadServiceMethod的时候还记得这句代码吗？`new ServiceMethod.Builder<>(this, method).build()`我们把retrofit传了进来，然后这里通过retrofit的callAdapter来获取ServiceMethod的adapter。下面是retrofit的源码获取c

```
 public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }


```

```
public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
   ·······省略的代码·······
   int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }

  ·······省略的代码·······
  }
  
```
我感觉只要不眼瞎都能看到这个serviceMethod的calladapter 是来自于我们retrofit之前进行初始化配置时候的`RxJavaCallAdapterFactory.create()`
其实我们可以在之前的图中看到这个flow，` .addCallAdapterFactory(RxJavaCallAdapterFactory.create())`最终指向的是ServiceMethod的CallAdapter。

进入`RxJavaCallAdapterFactory`的源码吧。其实这里的代码并不多，只不过有些杂乱。下面我会把不需要的代码过滤

```
public final class RxJavaCallAdapterFactory extends CallAdapter.Factory {
  /**
   * Returns an instance which creates synchronous observables that do not operate on any scheduler
   * by default.
   */
  public static RxJavaCallAdapterFactory create() {
    return new RxJavaCallAdapterFactory(null, false);
  }

  /**
   * Returns an instance which creates asynchronous observables. Applying
   * {@link Observable#subscribeOn} has no effect on stream types created by this factory.
   */
  public static RxJavaCallAdapterFactory createAsync() {
    return new RxJavaCallAdapterFactory(null, true);
  }

  /**
   * Returns an instance which creates synchronous observables that
   * {@linkplain Observable#subscribeOn(Scheduler) subscribe on} {@code scheduler} by default.
   */
  @SuppressWarnings("ConstantConditions") // Guarding public API nullability.
  public static RxJavaCallAdapterFactory createWithScheduler(Scheduler scheduler) {
    if (scheduler == null) throw new NullPointerException("scheduler == null");
    return new RxJavaCallAdapterFactory(scheduler, false);
  }

  private final @Nullable Scheduler scheduler;
  private final boolean isAsync;

  private RxJavaCallAdapterFactory(@Nullable Scheduler scheduler, boolean isAsync) {
    this.scheduler = scheduler;
    this.isAsync = isAsync;
  }

@Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
   ·······省略的代码·······

    if (isCompletable) {
      return new RxJavaCallAdapter(Void.class, scheduler, isAsync, false, true, false, true);
    }

   ·······省略的代码·······

    return new RxJavaCallAdapter(responseType, scheduler, isAsync, isResult, isBody, isSingle,
        false);
  }
}
```
可以看到这个RxJavaCallAdapterFactory继承了CallAdapter.Factory，实现了他的get方法，在get方法中我们就可以拿到这个CallAdapter,然后进行我们的adapt方法请求。
从上面的代码可以看到真正实现CallAdapter的是
`RxJavaCallAdapter` 我们继续看看这个RxJavaCallAdapter到底干了什么！！！同样我会过滤不需要的代码

```
final class RxJavaCallAdapter<R> implements CallAdapter<R, Object> {
·······省略的代码·······

   @Override public Object adapt(Call<R> call) {
    OnSubscribe<Response<R>> callFunc = isAsync
        ? new CallEnqueueOnSubscribe<>(call)
        : new CallExecuteOnSubscribe<>(call);

    OnSubscribe<?> func;
    if (isResult) {
      func = new ResultOnSubscribe<>(callFunc);
    } else if (isBody) {
      func = new BodyOnSubscribe<>(callFunc);
    } else {
      func = callFunc;
    }
    Observable<?> observable = Observable.create(func);

   ·······省略的代码·······
   
    return observable;
  }
}

```

好了，这下清晰了许多
***~~_~~这个RxJavaCallAdapter实现了CallAdapter的adapt请求。然后在adapt中需要传入参数就是Call 然而我们可以回到图中看到，OkHttpCall正好是Call的实现类。这里就是上图中的右边部分。~~_~~***
可以在这个方法中看到了，同步异步请求对应的不一样的然后调用不同的CallEnqueueOnSubscribe，CallExecuteOnSubscribe进行request
进入`CallEnqueueOnSubscribe`源码看看

```
final class CallEnqueueOnSubscribe<T> implements OnSubscribe<Response<T>> {
  private final Call<T> originalCall;

  CallEnqueueOnSubscribe(Call<T> originalCall) {
    this.originalCall = originalCall;
  }

  @Override public void call(Subscriber<? super Response<T>> subscriber) {
    // Since Call is a one-shot type, clone it for each new subscriber.
    Call<T> call = originalCall.clone();
    final CallArbiter<T> arbiter = new CallArbiter<>(call, subscriber);
    subscriber.add(arbiter);
    subscriber.setProducer(arbiter);

    call.enqueue(new Callback<T>() {
      @Override public void onResponse(Call<T> call, Response<T> response) {
        arbiter.emitResponse(response);
      }

      @Override public void onFailure(Call<T> call, Throwable t) {
        Exceptions.throwIfFatal(t);
        arbiter.emitError(t);
      }
    });
  }
}

```
看到了吧这里终于发起请求了可以看到，执行了OkHttpCall.enqueue请求，同时通过` arbiter.emitResponse(response);
`把参数回调了回来。

这就是retrofit的请求flow。

### 思路整理
1. 配置retrofit（addCallAdapterFactory）
2. 创建ApiService
3. 通过retrofit的create实例化ApiService接口
4. 创建ServiceMethod
5. 通过retrofit的AdapterFactories拿到ServiceMethod的CallAdapter
6. 创建OkHttpCall请求工具类
7. 通过ServiceMethod的CallAdapter的adapt进行请求并把第六步写好的call当参数传给adapt。
8. 把ServiceMethod请求回来的参数返回给对应的ApiService里面的方法
9. 请求完成

### ending
嘿嘿嘿，小伙伴们上面有什么错误的话或者不懂得都可以在留言中提及了，小编会在看到的第一时间响应。

***To Be Continued***


—— Cuieney 后记于 2017.03


