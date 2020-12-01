---
title: Activity 应用生命周期回调 ActivityLifecycleCallbacks 
date: 2019-09-09 23:17:26
tags: Activity
categories: Android
---

# Activity 应用生命周期回调 ActivityLifecycleCallbacks 

## ActivityLifecycleCallbacks 介绍

Application 类里面有一个接口 ActivityLifecycleCallbacks，它可以监听应用的 Activity 声明周期变化。

```java
    public interface ActivityLifecycleCallbacks {
        void onActivityCreated(Activity activity, Bundle savedInstanceState);
        void onActivityStarted(Activity activity);
        void onActivityResumed(Activity activity);
        void onActivityPaused(Activity activity);
        void onActivityStopped(Activity activity);
        void onActivitySaveInstanceState(Activity activity, Bundle outState);
        void onActivityDestroyed(Activity activity);
    }
```

可以看出接口提供了 7 个方法，除了常见的 6 个方法 （onCreated、onStarted、onResumed、onPaused、onStoped、onDestroyed），还有 1 个 onSaveInstanceState。

## 注册生命周期回调

Application 同时提供了注册和反注册接口的方法 registerActivityLifecycleCallbacks/unregisterActivityLifecycleCallbacks。

```java
    public void registerActivityLifecycleCallbacks(ActivityLifecycleCallbacks callback) {
        synchronized (mActivityLifecycleCallbacks) {
            mActivityLifecycleCallbacks.add(callback);
        }
    }
```

```java
    public void unregisterActivityLifecycleCallbacks(ActivityLifecycleCallbacks callback) {
        synchronized (mActivityLifecycleCallbacks) {
            mActivityLifecycleCallbacks.remove(callback);
        }
    }
```

```java
    private ArrayList<ActivityLifecycleCallbacks> mActivityLifecycleCallbacks =
            new ArrayList<ActivityLifecycleCallbacks>();
```

可以看出就是将接口添加到接口列表中，或者将接口从接口列表移除。2 个方法内部都用 synchronized 关键字对添加和删除操作加锁，保证线程同步。

## 回调消息分发

Application 内部有一些 dispatchActivityXXX 方法，用来回调分发 Activity 声明周期回调的消息。

dispatchActivityCreated 的代码如下：

```java
    /* package */ void dispatchActivityCreated(Activity activity, Bundle savedInstanceState) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i=0; i<callbacks.length; i++) {
                ((ActivityLifecycleCallbacks)callbacks[i]).onActivityCreated(activity,
                        savedInstanceState);
            }
        }
    }
```

可以看出这个方式是 package 访问级别修饰，只有同一个包里面的类可以访问。

dispatchActivityCreated 将 activity 和 savedInstanceState 分发给在 Application 里面注册的每个接口。

dispatchActivityCreated 会被 Activity 类调用，并传递 Activity 自身作为参数。

Activity 的 onCreate 方法如下：

```java
    @CallSuper
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        ...
        mFragments.dispatchCreate();
        getApplication().dispatchActivityCreated(this, savedInstanceState);
        if (mVoiceInteractor != null) {
            mVoiceInteractor.attachActivity(this);
        }
        mRestoredFromBundle = savedInstanceState != null;
        mCalled = true;
    }
```

可以看出 onCreate 调用了 getApplication() 获取 Application，然后调用 dispatchActivityCreated 将 activity 自身 和 savedInstanceState 作为参数传递。

## 总结

1. 在 Application 注册的接口可以收到 Activity 的声明周期回调。
2. ActivityLifecycleCallbacks 只能收到本应用的 activity 回调，因为回调都是由本应用 activity 的声明周期传递过来的。