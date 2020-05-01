# RxJava 结合 OkHttp 访问网络

这里以访问 Github api 为例。

## 创建 Observable

首先使用 create 方法创建 Observable，在回调方法中使用 OkHttp 异步请求网络，然后将返回结果发送给 emitter 的 onNext 方法。同时实现 emitter 的 onError 和 onComplete。

```java
private Observable<String> getObservable() {
        return Observable.create(emitter -> {
            OkHttpClient okHttpClient = new OkHttpClient();
            Request request = new Request.Builder()
                    .url("https://api.github.com/users/flutter")
                    .get()
                    .build();
            Call call = okHttpClient.newCall(request);
            call.enqueue(new Callback() {
                @Override
                public void onFailure(@NotNull Call call, @NotNull IOException e) {
                    emitter.onError(e);
                }

                @Override
                public void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {
                    String str = response.body().string();
                    Log.d(TAG, "onResponse:" + response + ", body:" + str);
                    emitter.onNext(str);
                    emitter.onComplete();
                }
            });
        });
    }
```

## 订阅观察者 Observer

观察者订阅代码如下：

```java
    private void getAsyncHttp() {
        Observable<String> observable = getObservable();
        observable.subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(str -> Toast.makeText(getApplicationContext(), "load success:" + str, Toast.LENGTH_LONG).show(),
                        throwable -> Log.e(TAG, "onError:" + throwable.getMessage()), () -> Log.d(TAG, "onCompleted"));
    }
```

观察者收到 onNext 消息后发送 Toast。因为使用了 observeOn 方法运行在 Android 主线程，所以可以做 UI 操作。

输出结果如下：

```
RxJavaActivity: onResponse:Response{protocol=http/1.1, code=200, message=OK, url=https://api.github.com/users/flutter}, body:{"login":"flutter","id":14101776, ...
```

## 添加网络权限

网络请求需要在 AndroidManifest 添加网络权限

```
    <uses-permission android:name="android.permission.INTERNET" />
```

如果是 P 版本以上的非 Https 请求，需要在 application 节点配置允许明文。

```
        android:networkSecurityConfig="@xml/network_security_config"
```

res/xml 目录的 network_security_config.xml

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true" />
</network-security-config>
```
