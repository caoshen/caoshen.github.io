---
title: Android 源码的单例模式
date: 2019-10-23 01:28:01
tags: 设计模式
categories: Android
---

# Android 源码的单例模式

单例模式可以保证在内存中只有一个实例，可以避免对象频繁地创建与消耗。

下面以 android 29 为例，看 2 种单例的实现。

## SystemServiceRegistry 的 LayoutInflater

Context 的实现类 ContextImpl 中保存了所有 Service 本地代理的缓存。常用的 Context 的 getSystemService 方法就是从  SystemServiceRegistry 中获取 Service。因为缓存在一个 ArrayMap 中，所以每次 get 到的都是同一个实例，这样就实现了单例模式。

SystemServiceRegistry 类的实现如下：

```java
final class SystemServiceRegistry {
    private static final String TAG = "SystemServiceRegistry";

    // Service registry information.
    // This information is never changed once static initialization has completed.
    private static final Map<Class<?>, String> SYSTEM_SERVICE_NAMES =
            new ArrayMap<Class<?>, String>();
    private static final Map<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
            new ArrayMap<String, ServiceFetcher<?>>();
    private static int sServiceCacheSize;

    // Not instantiable.
    private SystemServiceRegistry() { }

    static {
        ...
        registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});
        ...
```

可以看出每次拿到的 LayoutInflater 是 SYSTEM_SERVICE_FETCHERS 中保存的 PhoneLayoutInflater。

## ActivityManager 的 IActivityManagerSingleton

ActivityManager 的 getService 方法会返回一个 IActivityManagerSingleton 单例。

```java
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }
```

IActivityManagerSingleton 是一个常量，虽然它没有全部大写。

```java
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

可以看出它是一个 Sington 类型。

```java
public abstract class Singleton<T> {
    @UnsupportedAppUsage
    private T mInstance;

    protected abstract T create();

    @UnsupportedAppUsage
    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}
```

Singleton 是一个懒汉单例模式的抽象类，暴露 create 方法用来产生单例。这一点和常用的构造方法 new 出来一个单例有所不同。