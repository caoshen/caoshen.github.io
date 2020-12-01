---
title: Retrofit 源码分析
date: 2019-08-20 00:42:10
tags: Android
categories: Android
---

# Retrofit 源码分析

这里以 retrofit2.6.1 版本为例，从 3 个方面分析 Retrofit 的源代码：

1. Retrofit 创建；
2. Call 创建；
3. Call 入队（enqueue）.

## Retrofit 的创建

使用 Retrofit 的时候，首先会用建造者模式构造 Retrofit。

```java
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(IP_TAOBAO_SERVICE)
                .addConverterFactory(GsonConverterFactory.create())
                .build();
```

### Platform 构造

Builder 的无参构造方法调用了 Platform.get()。获取一个 Platform，然后作为参数传递给构造方法。

```java
    public Builder() {
      this(Platform.get());
    }
```

Platform 的 get 方法是单例模式，返回 Platform 的单例。

```java
  private static final Platform PLATFORM = findPlatform();

  static Platform get() {
    return PLATFORM;
  }

  private static Platform findPlatform() {
    try {
      Class.forName("android.os.Build");
      if (Build.VERSION.SDK_INT != 0) {
        return new Android();
      }
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("java.util.Optional");
      return new Java8();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
  }
```

可以看出采用了饿汉式单例模式，直接获取单例然后复制给静态常量。findPlatform 方法按照 Android、Java8、默认 Platform 的顺序依次尝试根据不同的平台获取不同的 Platform。判断平台的方法是用 Class.formName 反射查找对应平台的类名。

### build 方法

Retrofit 构造时最后会调用 build 方法。

```java
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
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(
          1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);
      converterFactories.addAll(platform.defaultConverterFactories());

      return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
    }
  }
```

可以看出首先判断 baseUrl，如果 baseUrl 为空，直接抛出异常。

如果没有设置 callFactory，默认使用 OkHttpClient。可见 Retrofit 是在 OkHttp 的基础上构建的，是对 OkHttp 的封装。

如果没有设置 callbackExecutor，使用 platform 默认的 defaultCallbackExecutor。如果是 Android 的 platform，就是主线程 handler 构造的 executor，代码如下：

```java
    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
```
可以看出 Android 的 defaultCallbackExecutor 实现了 Executor 接口，本质上把 Runnable post 到主线程的 Handler 执行。 

随后设置了 callAdapterFactories，用来将 Call 转化为实际的网络请求实现。

```java
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
```

接着设置数据转化类 （Converter）。Converter 有 3 种类型：

1. 内置的 Converter（BuiltInConverters）；
2. 外部设置的 Converter （converterFactories）；
3. 平台默认的 Converter（platform.defaultConverterFactories()）。

```java
      List<Converter.Factory> converterFactories = new ArrayList<>(
          1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);
      converterFactories.addAll(platform.defaultConverterFactories());
```

最后会把以上所有的构造参数传递给 Retrofit 的构造方法并返回一个 Retrofit 对象。

```java
      return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
```

## Call 的创建

Retrofit 调用 create 方法，将类（一般是接口类）转换为代理类对象。

```java
  public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];

          @Override public @Nullable Object invoke(Object proxy, Method method,
              @Nullable Object[] args) throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
          }
        });
  }
```

可以看出 create 方法返回了 Proxy.newInstance 代理对象。当调用接口的方法时，实际上调用了 InvocationHandler 的 invoke 方法。

invoke 方法有 3 个参数，代理对象 proxy、调用方法 method、方法参数 args。

invoke 方法最后调用了 loadServiceMethod 方法。

### 获取 ServiceMethod

loadServiceMethod 会从缓存的 concurrentHashMap 中得到 Method 对应的 serviceMethod。

```java
  ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

可以看出如果 serviceMethodCache 中存在 ServiceMethod，直接返回；否则调用 ServiceMethod 的 parseAnnotations 方法返回一个 ServiceMethod，并 put 到 serviceMethodCache 中。

ServiceMethod 的 parseAnnotations 方法如下：

```java
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(method,
          "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }

    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
```

可以看出 ServiceMethod 的 parseAnnotations 方法最后调用的是 HttpServiceMethod 的 parseAnnotations 方法。

```java
  static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    ...

    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
    ...
    if (!isKotlinSuspendFunction) {
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    }
    ...
  }
```

可以看出 HttpServiceMethod 的 parseAnnotations 方法最后调用了 createCallAdapter，并且构造了一个 CallAdapted 对象。

CallAdapted 是 HttpServiceMethod 的内部类，同时继承了 HttpServiceMethod。

```java
  static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
    ...

    @Override protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
      return callAdapter.adapt(call);
    }
  }
```

可以看出 CallAdapted 复写的 adapt 方法调用了 callAdapter 的 adapt 方法。而 callAdapter 是 createCallAdapter 的返回值。

查看 createCallAdapter 方法，它调用了 retrofit 的 callAdapter，同时传入了 returnType 和 annotations (返回值和注解)。

```java
  private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
      Retrofit retrofit, Method method, Type returnType, Annotation[] annotations) {
    try {
      //noinspection unchecked
      return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
      throw methodError(method, e, "Unable to create call adapter for %s", returnType);
    }
  }
```

callAdapter 方法实际调用了 nextCallAdapter 方法。

```java
  public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
...
    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
...
  }

```

可以看出 nextCallAdapter 会从 callAdapterFactories 里面获取当前的 callAdapterFactory，并且 get 一个 CallAdapter。

### defaultCallAdapterFactories（默认调用适配器工厂）

之前 retrofit 构建时，会根据不同的 Platform 传递不同的默认 CallAdapterFactory。

查看 Android Platform 的 CallAdapterFactory。

```java
    @Override List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
        @Nullable Executor callbackExecutor) {
      if (callbackExecutor == null) throw new AssertionError();
      DefaultCallAdapterFactory executorFactory = new DefaultCallAdapterFactory(callbackExecutor);
      return Build.VERSION.SDK_INT >= 24
        ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
        : singletonList(executorFactory);
    }
```

可以看出 SDK 版本大于等于 24 （Android N 版本）时，有 2 个 CallAdapterFacotry。

1. CompletableFutureCallAdapterFactory.INSTANCE；
2. DefaultCallAdapterFactory。

查看 DefaultCallAdapterFactory 的 get 方法：

```java
  @Override public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    if (!(returnType instanceof ParameterizedType)) {
      throw new IllegalArgumentException(
          "Call return type must be parameterized as Call<Foo> or Call<? extends Foo>");
    }
...
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return executor == null
            ? call
            : new ExecutorCallbackCall<>(executor, call);
      }
    };
  }
```

可以看出 get 方法首先判断了返回类型是不是 Call 类型，如果不是，直接 return。
最后返回了一个封装了回调 executor 和 call 的 ExecutorCallbackCall，回调方法将在 executor 中执行。

回到 HttpServiceMethod 类，查看它的 invoke 方法：

```java
  @Override final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }
```

可以看出调用方法之前，先用 requestFactory、args 等参数构造了一个 OKHttpCall，最后交给上面提到的 defaultCallAdapter 执行。

这里传入的 requestFactory 就是 Retrofit 里面的 RequestFactory 类对象。在 RequestFactory 类里面通过建造者模式构造了真正的 OkHttp 请求。

回到 ServiceMethod 类的 parseAnnotations 方法：

```java
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
...
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
```

可以看出第一步就是构造一个 RequestFactory。剩余的操作交给了 HttpServiceMethod.parseAnnotations 方法，它可以指定 callAdapter 和 responseConverter。

### 为什么 Retrofit 的回调在主线程

之前提到 Retrofit 的回调在主线程。查看 Android Platform 的代码：

```java
  static class Android extends Platform {
    ...
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }
...
    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
```
可以看出它封装了主线程 Handler 作为 MainThreadExecutor。

查看 DefaultCallAdapterFactory 类的 enqueue 方法，就是封装了 runnable 交给 callbackExecutor 执行。

```java
    @Override public void enqueue(final Callback<T> callback) {
     ...
      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }
...
    }
```

可以发现，回调的 runnable 都 post 到主线程的 Looper 构造的 Handler，所以回调消息都在主线程执行。

## Call 的 enqueue 方法

上一节分析了 ExecutorCallbackCall ，Call 的 enqueue 方法调用的是 ExecutorCallbackCall 的 enqueue 方法。

ExecutorCallbackCall 的 enqueue 方法调用了 delegate 的 enqueue 方法。
这里的 delegate 是 OkHttpCall。

```java
  @Override public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    okhttp3.Call call;
   ...

    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        ...
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        callFailure(e);
      }
...
  }
```
可以看出 OkHttpCall 的 enquene 方法就是 okhttp3.Call 的 enqueue 方法。

onResponse 回调中调用了 parseResponse 方法。

```java
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          throwIfFatal(e);
          callFailure(e);
          return;
        }

        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          throwIfFatal(t);
          t.printStackTrace(); // TODO this is not great
        }
      }
```

OkHttpCall 的 parseResponse 方法如下：

```java
  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
    ...
    try {
      T body = responseConverter.convert(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```

可以看出调用 responseConverter 转化 catchingBody 为指定类型的对象。
这里的 responseConverter 是 HttpServiceMethod 的 createResponseConverter 方法返回的，其实就是 Retrofit 构造是传入的 Converter 对象，比如 GsonConverterFactory。

GsonConverterFactory 会通过 responseBodyConverter 方法构造 GsonResponseBodyConverter。GsonResponseBodyConverter 的 convert 方法如下：

```java
  @Override public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      T result = adapter.read(jsonReader);
      if (jsonReader.peek() != JsonToken.END_DOCUMENT) {
        throw new JsonIOException("JSON document was not fully consumed.");
      }
      return result;
    } finally {
      value.close();
    }
  }
```

可以看出 convert 方法用 gson 的 newJsonReader 方法构造 JsonReader，将网络响应结果转换为定义的 Json 对象。

## 总结

Retrofit 对 OkHttp 进行了封装，它使用接口定义和注解的方式使得代码更为简洁，本质上是 Retrofit 将网络请求接口转换为具体的 OkHttpCall 对象，封装了 HttpServiceMethod、回调接口、Gson 转化类，最后通过 OkHttp 的 enqueue 方法完成网络请求。