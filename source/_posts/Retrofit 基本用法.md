# Retrofit 基本用法

## Retrofit 介绍

Retrofit 是 Square 开发的 Android 网络请求框架，它是基于 OkHttp 实现的。与其他网络框架不同，它使用运行时注解提供功能。

## Retrofit 依赖

在 build.gradle 配置依赖如下：

```
    implementation 'com.squareup.retrofit2:retrofit:2.6.1'
    implementation 'com.squareup.retrofit2:converter-gson:2.6.1'
```

第一行配置了 retrofit 依赖。第二行配置了 gson 转换依赖。

## Retrofit 注解分类

Retrofit 有 3 类注解：

1. 请求方法注解；
2. 标记类注解；
3. 参数类注解。

Http 请求方法注解有 8 种：GET、POST、PUT、DELETE、HEAD、PATCH、OPTIONS、HTTP。

标记类注解有 3 种：FormUrlEncoded、Multipart、Streaming。

参数类注解：Header、Headers、Body、Path、Field、FieldMap、Part、PartMap、Query 和 QueryMap 等。

## GET 请求访问网络

这里以淘宝 IP 库为例。
首先定义网络请求接口

```java
public interface IpService {
    @GET("getIpInfo.php?ip=59.108.54.37")
    Call<IpModel> getIpMsg();
}
```

通过建造者模式构建 Retrofit，使用 Retrofit 动态代理获取定义的 Service 接口，并调用接口的 getIpMsg 方法获取 Call 对象。

```java
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(url)
                .addConverterFactory(GsonConverterFactory.create())
                .build();
        IpService ipService = retrofit.create(IpService.class);
        Call<IpModel> call = ipService.getIpMsg();
```

然后使用 call.enqueue 方法异步请求网络。

```java
        call.enqueue(new Callback<IpModel>() {
            @Override
            public void onResponse(Call<IpModel> call, Response<IpModel> response) {
                
            }

            @Override
            public void onFailure(Call<IpModel> call, Throwable t) {

            }
        });
```

这里的 Callback 默认运行在 UI 线程。

如果想同步请求网络，使用 execute 方法。

如果想取消请求，使用 cancel 方法。

## POST 请求访问网络

首先用 @FormUrlEncoded 注解表示它是一个表单请求。然后再 getIpMsg 方法使用 @Field 注解表明参数 String 的键，从而组成一组键值对进行传递。

```java
public interface IpServicePost {
    
    @FormUrlEncoded
    @POST("getIpInfo.php")
    Call<IpModel> getIpMsg(@Field("ip") String first);
}
```

Post 请求网络的代码如下：

```java
    private void doPost() {
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(IP_TAOBAO_SERVICE)
                .addConverterFactory(GsonConverterFactory.create())
                .build();
        IpServicePost ipServicePost = retrofit.create(IpServicePost.class);
        Call<IpModel> call = ipServicePost.getIpMsg(IP);
        call.enqueue(new Callback<IpModel>() {
            @Override
            public void onResponse(Call<IpModel> call, Response<IpModel> response) {

            }

            @Override
            public void onFailure(Call<IpModel> call, Throwable t) {

            }
        });
    }
```

## 消息报头 Header

在 HTTP 请求中，为了防止攻击或者过滤不安全的访问，或是添加特殊加密的访问标记时，会在消息报头携带特殊的消息头处理。

Retrofit 提供了 @Header 注解添加消息报头。有 2 种添加方式：

1. 静态添加；
2. 动态添加。

### 静态方式

```java
public interface HeaderStaticService {

    @GET("some/endpoint")
    @Headers("Accept-Encoding: application/json")
    Call<ResponseBody> getCarType();
}
```

如果添加多个消息报头，可以使用大括号{}。

```java
public interface HeaderStaticMultiService {

    @GET("some/endpoint")
    @Headers({
            "Accept-Encoding: application/json",
            "User-Agent: MoonRetrofit"
    })
    Call<ResponseBody> getCarType();
}
```

### 动态方式

```java
public interface HeaderDynamicService {

    @GET("some/endpoint")
    Call<ResponseBody> getCarType(@Header("Location") String location);
}
```

使用 @Header 注解，可以通过调用 getCarType 方法动态地添加消息报头。

## 总结

Retrofit 使用注解的方式提供网络请求功能。使用 @GET 、@POST 方法访问网络，使用 @Header 添加消息报头。
