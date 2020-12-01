---
title: RxJava 结合 Retrofit 访问网络
date: 2019-09-01 10:59:09
tags: Android
categories: Android
---

# RxJava 结合 Retrofit 访问网络

RxJava 可以配合 Retrofit 访问网络，这里以 Github API 为例。

## 使用前配置依赖

配置 build.gradle

```
    // retrofit
    implementation 'com.squareup.retrofit2:retrofit:2.6.1'
    implementation 'com.squareup.retrofit2:converter-gson:2.6.1'
    implementation 'com.squareup.retrofit2:adapter-rxjava2:2.6.1'
    // rxjava
    implementation "io.reactivex.rxjava2:rxjava:2.2.11"
    implementation 'io.reactivex.rxjava2:rxandroid:2.1.1'
```

RxJava2 是为了使用 RxJava2 转换器，即 RxJava2CallAdapterFactory。

注意如果用的 RxJava2，就使用 adapter-rxjava2，否则是 adapter-rxjava。

```
    implementation 'com.squareup.retrofit2:adapter-rxjava2:2.6.1'
```

## 网络请求接口

Retrofit 的请求接口返回的是 Call。如果结合 RxJava 使用，需要把 Call 改为 Observable。

接口代码如下：

```java
public interface RetroService {

    @GET("flutter")
    Observable<RetroData> getData();
}
```

## 网络请求方法

RxJava 配合 Retrofit 网络请求代码如下：

```java
    private void getData() {
        String url = "https://api.github.com/users/";
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(url)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();
        RetroService retroService = retrofit.create(RetroService.class);
        disposable = retroService.getData()
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(data -> Log.d(TAG, "onNext:" + data.created_at),
                        throwable -> Log.e(TAG, "onError:" + throwable.getMessage()),
                        () -> Log.d(TAG, "onComplete")
                );
    }
```

调用 addCallAdapterFactory(RxJava2CallAdapterFactory.create()) 可以使之前定义的接口返回 Observable 而不是 Call。这样就能 RxJava 提供的方法来对网络请求结果做处理。

## 请求返回数据格式

返回数据 RetroData 的格式如下：

```java
public class RetroData {
    public String login;
    public long id;
    public String node_id;
    public String avatar_url;
    public String gravatar_id;
    public String url;
    public String html_url;
    public String followers_url;
    public String following_url;
    public String gists_url;
    public String starred_url;
    public String subscriptions_url;
    public String organizations_url;
    public String repos_url;
    public String events_url;
    public String received_events_url;
    public String type;
    public String site_admin;
    public String name;
    public String company;
    public String blog;
    public String location;
    public String email;
    public String hireable;
    public String bio;
    public long public_repos;
    public long public_gists;
    public long followers;
    public long following;
    public String created_at;
    public String updated_at;

    @Override
    public String toString() {
        return "RetroData{" +
                "login='" + login + '\'' +
                ", id=" + id +
                ", node_id='" + node_id + '\'' +
                ", avatar_url='" + avatar_url + '\'' +
                ", gravatar_id='" + gravatar_id + '\'' +
                ", url='" + url + '\'' +
                ", html_url='" + html_url + '\'' +
                ", followers_url='" + followers_url + '\'' +
                ", following_url='" + following_url + '\'' +
                ", gists_url='" + gists_url + '\'' +
                ", starred_url='" + starred_url + '\'' +
                ", subscriptions_url='" + subscriptions_url + '\'' +
                ", organizations_url='" + organizations_url + '\'' +
                ", repos_url='" + repos_url + '\'' +
                ", events_url='" + events_url + '\'' +
                ", received_events_url='" + received_events_url + '\'' +
                ", type='" + type + '\'' +
                ", site_admin='" + site_admin + '\'' +
                ", name='" + name + '\'' +
                ", company='" + company + '\'' +
                ", blog='" + blog + '\'' +
                ", location='" + location + '\'' +
                ", email='" + email + '\'' +
                ", hireable='" + hireable + '\'' +
                ", bio='" + bio + '\'' +
                ", public_repos=" + public_repos +
                ", public_gists=" + public_gists +
                ", followers=" + followers +
                ", following=" + following +
                ", created_at='" + created_at + '\'' +
                ", updated_at='" + updated_at + '\'' +
                '}';
    }
}
```

对于日常开发来说，返回的错误码（code）和错误描述（message）一般是固定的几种，但是返回的数据类型（data）可能是不断变化的。

对于可能变化的返回数据类型，可以使用 code 加上数据泛型的方式处理。

```java
public class HttpResult<T> {
    private int code;
    private T data;

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }
}
```

## 取消请求

如果异步任务耗时较长，需要在 Activity 的 onDestroy 中取消请求，防止内存泄漏。

```java
    @Override
    protected void onDestroy() {
        if (disposable != null && !disposable.isDisposed()) {
            disposable.dispose();
        }
        super.onDestroy();
    }
```

disposable 是 RxJava subscribe 方法的返回值，可以调用 dispose 方法取消。

如果有多个网络请求，产生了多个 disposable，可以使用 CompositeDisposable 同时取消。

```java
        CompositeDisposable compositeDisposable = new CompositeDisposable();
        compositeDisposable.add(disposable);
```

以上代码将 disposable 加入到 CompositeDisposable 中。

```java
    @Override
    protected void onDestroy() {
        if (compositeDisposable != null) {
            compositeDisposable.dispose();
        }
        super.onDestroy();
    }
```
以上代码可以取消多个 disposable。