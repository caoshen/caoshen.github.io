---
title: Android 源码的代理模式
date: 2019-11-18 23:57:30
tags: 设计模式
categories: Android
---

# Android 源码的代理模式

## 代理模式介绍

代理模式（Proxy Pattern）也称为委托模式，是结构型设计模式的第一个模式。

## 代理模式的定义

为其他对象提供一种代理以控制对这个对象的访问。

## 代理模式的使用场景

当无法或不想直接访问某个对象或访问某个对象存在困难时，可以通过一个代理对象来间接访问，为了保证客户端使用的透明性，委托对象与代理对象需要实现相同的接口。

## Android 源码中的代理模式实现

当应用通过 ActivityManager 访问 ActivityManagerService 的接口方法时，本质上是通过 IActivityManager 生成的代理对象访问的，是一种典型的代理模式。

与自定义的 aidl 文件类似，IActivityManager.java 也是通过 IActivityManager.aidl 生成的，只不过 IActivityManager.aidl 是系统 Framework 定义的 aidl 文件。

### IActivityManager

IActivityManager 位于 frameworks/base/core/java/android/app/IActivityManager.aidl。

IActivityManager 接口定义如下：

```aidl
...
/**
 * System private API for talking with the activity manager service.  This
 * provides calls from the application back to the activity manager.
 *
 * {@hide}
 */
interface IActivityManager {
    ...
    @UnsupportedAppUsage
    int startActivity(in IApplicationThread caller, in String callingPackage, in Intent intent,
            in String resolvedType, in IBinder resultTo, in String resultWho, int requestCode,
            int flags, in ProfilerInfo profilerInfo, in Bundle options);
    ...
}
```

### ActivityManagerService

接口真正的实现都在 ActivityManagerService 类。

```java
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    ...
}
```

可以看出 ActivityManagerService 继承了 IActivityManager.Stub 类。IActivityManager.Stub 是 IActivityManager.aidl 生成的 java 文件的内部抽象类。


ActivityManagerService 实现的 startActivity 方法如下：

```java
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
```

## ActivityManager getService

ActivityMangerService 的本地代理是 ActivityManager 通过 getService() 方法得到的 IActivityManager 对象。

```java
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```

首先通过 IActivityManagerSingleton 的 get 方法获取一个 IActivityManager 的单例。IActivityManager 通过 ServiceManager 的 getService 方法得到代理的 IBinder 对象，然后通过 IActivityManager.Stub.asInterface 方法将代理的 IBinder 转化为 IActivityManager 接口。

因此，使用 IActivityManager 接口就像直接调用 ActivityMangerService 一样。但实际上，调用代理 IActivityManager 时，会将数据打包跨进程传递给 Server 端的 ActivityMangerService 处理并返回结果。这些过程都在 IActivityManager.java 的内部实现中完成，所以看上去调用代理时隐藏了内部 transact 的细节。