---
title: AIDL
date: 2019-02-25 15:39:25
tags:
---

# AIDL

AIDL(Android Interface Definition Language)Android 接口描述语言，故名思意，使用来定义接口的。创建 AIDL 文件后，编译会自动产生对应的 Java 接口以及实现接口通信的样板代码。

```
// IMyAidlInterfaceB.aidl
package com.example.b.myapplicationb;

// Declare any non-default types here with import statements

interface IMyAidlInterfaceB {
    String getSomeData();
}
```

## AIDL 支持的数据类型

基本数据类型：int short long float double char boolean byte;

String 和 CharSequence;

List: 只支持 ArrayList, List 的每个元素必须被 AIDL 支持；

Map: 只支持 HashMap, HashMap 的每个元素必须被 AIDL 支持，包括 key 和 value；

Parcelable：所有实现了 Parcelable 的对象;

AIDL 接口本身。

1. Parcelable 就算在同一个 package，也必须使用 import 语句；
2. 如果 AIDL 中用到了自定义的 Parcelable 对象，必须为它创建 AIDL 文件来声明 Parcelable。


## AIDL 参数方向

AIDL 除了基本数据类型，其他的参数有 in out inout 三种参数传递方向，默认是 in 方向。

如果是 in 方向，表示参数从客户端到服务端传递时，服务端会接收参数，但是服务端接收到参数后不能改变参数。

如果是 out 方向，表示参数从客户端到服务端传递时，服务端不接收参数，而是自己 new 一个参数对象。

如果是 inout 方向，表示参数从客户端到服务端传递时，服务端会接收参数，也可以改变参数。

## AIDL oneway

如果 AIDL 文件的接口定义以 oneway 开头，说明这个接口方法是单向接口，客户端发送请求后不会同步等待，而是立即返回。

## RemoteCallbackList

使用 AIDL 通信时，如果服务端需要保存多个客户端注册的接口回调，不能使用 List 保存，而是使用 RemoteCallbackList。

因为跨进程传输是一个序列化和反序列化的过程，这就导致了客户端发送的接口回调和服务端收到的回调其实是两个不同的对象。如果只是通过判断对象来反注册就会找不到最开始注册的回调。

RemoteCallbackList 类里面有个 ArrayMap 专门保存回调。它的 key 是 IBinder，它的值是 Callback。Callback 由 IInterface 和 cookie 组成。cookie 是一个 Object，一般用来保存一些额外信息，比如调用方的 uid。

```java
public class RemoteCallbackList<E extends IInterface>
```

```java
ArrayMap<IBinder, Callback> mCallbacks = new ArrayMap<IBinder, Callback>();
```

当服务端通知回调时，使用 beginBroadcast/finishiBroadcast来开始和结束回调，使用 getBroadcastItem(i) 来获取每个回调。