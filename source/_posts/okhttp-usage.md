---
title: Okhttp 基本用法
date: 2019-08-20 00:40:39
tags: Android
categories: Android
---

# Okhttp 基本用法

Okhttp 是一个 Android 常用的开源网络框架。Github 地址如下：

[Okhttp](https://github.com/square/okhttp)

OkHttp 当前的版本是 4.0.1，现在同时支持 Java 和 Kotlin。

## HttpURLConnection 和 OkHttp

从 Android 4.4 版本开始，系统内置了 OkHttp。从 URL 类的 createBuiltinHandler 方法可以发现构造 Handler 时反射调用了 Okhttp 的 HttpHandler。其中 okhttp 的包名是 com.android，这是因为系统在依赖的时候替换了原来的 com.squareup 包名。

```java
    private static URLStreamHandler createBuiltinHandler(String protocol)
            throws ClassNotFoundException, InstantiationException, IllegalAccessException {
        URLStreamHandler handler = null;
        if (protocol.equals("file")) {
            handler = new sun.net.www.protocol.file.Handler();
        } else if (protocol.equals("ftp")) {
            handler = new sun.net.www.protocol.ftp.Handler();
        } else if (protocol.equals("jar")) {
            handler = new sun.net.www.protocol.jar.Handler();
        } else if (protocol.equals("http")) {
            handler = (URLStreamHandler)Class.
                    forName("com.android.okhttp.HttpHandler").newInstance();
        } else if (protocol.equals("https")) {
            handler = (URLStreamHandler)Class.
                    forName("com.android.okhttp.HttpsHandler").newInstance();
        }
        return handler;
    }
```

## 依赖引用

Okhttp 依赖了 Okio 和 Kotlin 标准库。

使用的时候直接依赖 Okhttp 的最新版本。

```
implementation("com.squareup.okhttp3:okhttp:4.0.1")
```

## 异步请求

### GET 请求

GET 请求 CSDN 博客地址，代码如下：

```java
    private void doGet() {
        Request.Builder builder = new Request.Builder()
                .url("http://blog.csdn.net")
                .method("GET", null);
        final Request req = builder.build();
        OkHttpClient okHttpClient = new OkHttpClient();
        Call call = okHttpClient.newCall(req);
        call.enqueue(new Callback() {
            @Override
            public void onFailure(@NotNull Call call, @NotNull IOException e) {
                Log.e(TAG, "onFailure");
            }

            @Override
            public void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {
                Log.d(TAG, "onResponse:" + response.body().toString());
            }
        });
    }
```

主要分 3 步：

1. 通过 Request Builder 构造 Request
2. 通过 OkHttpClient 和 Request 构造 Call
3. Call 入队

值得注意的是，Callback 的回调方法不主线程，注意不要直接做更新 UI 的操作。

### POST 请求

POST 请求淘宝 IP 地址库，代码如下：

```java
    private void doPost() {
        RequestBody requestBody = new FormBody.Builder()
                .add("ip", "59.108.54.37")
                .build();
        Request.Builder builder = new Request.Builder()
                .url("http://ip.taobao.com/service/getIpInfo.php")
                .post(requestBody);
        final Request req = builder.build();
        OkHttpClient okHttpClient = new OkHttpClient();
        Call call = okHttpClient.newCall(req);
        call.enqueue(new Callback() {
            @Override
            public void onFailure(@NotNull Call call, @NotNull IOException e) {
                Log.e(TAG, "onFailure");
            }

            @Override
            public void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {
                Log.d(TAG, "onResponse:" + response.body().toString());
            }
        });
    }
```

POST  请求需要在 Request 里面 post 添加 FormBody。

<!-- ### 上传文件

### 下载文件

### 上传 MultiPart 文件

### 超时与缓存

### 取消请求 -->

## 同步请求

如果是同步请求，直接调用 execute 方法。代码如下：

```java
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  try (Response response = client.newCall(request).execute()) {
    return response.body().string();
  }
}
```

## 封装 Okhttp

为了可重用和可扩展，一般需要对 Okhttp 进行封装。

封装后的 Okhttp 库主要解决 2 个问题：

1. 避免重复代码调用；
2. 请求结果回调到 UI 线程。

这里推荐使用开源的 OKHttpFinal。Github 地址如下：

[OkHttpFinal](https://github.com/pengjianbo/OkhttpFinal)