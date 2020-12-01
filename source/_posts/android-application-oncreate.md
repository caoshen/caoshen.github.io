---
title: Application 的 onCreate 和 attachBaseContext
date: 2019-12-01 23:57:54
tags: 四大组件
categories: Android
---

# Application 的 onCreate 和 attachBaseContext

Application 的 onCreate 和 attachBaseContext 是 Application 的两个回调方法，通常我们会在其中做一些初始化操作。

## onCreate 和 attachBaseContext 顺序

Application 的 attachBaseContext 在 onCreate 之前执行。

## handleBindApplication

App 的 application 创建是在 ActivityThread 的 handleBindApplication 方法完成的。

```java
    private void handleBindApplication(AppBindData data) {
        ...
        // If the app is Honeycomb MR1 or earlier, switch its AsyncTask
        // implementation to use the pool executor.  Normally, we use the
        // serialized executor as the default. This has to happen in the
        // main thread so the main looper is set right.
        if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
            AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
        }

        ...
        final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
        ...
            app = data.info.makeApplication(data.restrictedBackupMode, null);

            ...

            mInitialApplication = app;

            ...
            try {
                mInstrumentation.callApplicationOnCreate(app);
            ...
    }
```

handleBindApplication 通过 LoadedApk 的 makeApplication 构造 application。

LoadedApk 的 makeApplication 方法如下：

```java
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        ...
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        ...
        return app;
    }
```

makeApplication 通过 ContextImpl.createAppContext 构造了 contextImpl，然后用 newApplication 构造了 application。

Instrumentation 的 newApplication 方法如下：

```java
    public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = getFactory(context.getPackageName())
                .instantiateApplication(cl, className);
        app.attach(context);
        return app;
    }
```

可以看出 newApplication 先构造了 Application，然后再 attach context，这一点和 Activity 的构造类似，都是先创建，再 attach 一些内容。

Application 的 attach 方法如下：

```java
    /* package */ final void attach(Context context) {
        attachBaseContext(context);
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
    }
```

先调用了 attachBaseContext，然后给 mLoadedApk 赋值。

回到 ActivityThread 的 handleBindApplication，首先会在 makeApplication 时回调 attachBaseContext，然后执行 callApplicationOnCreate。


Instrumentation 的 callApplicationOnCreate 方法如下：

```java
    public void callApplicationOnCreate(Application app) {
        app.onCreate();
    }
```

callApplicationOnCreate 回调了 Application 的 onCreate 方法。

## 总结

Application 的 attachBaseContext 在 onCreate 之前执行。attachBaseContext 中拿到了创建的 ContextImpl，但是此时 Application 没有创建完成，比如 mLoadedApk 的赋值还没有执行。此时如果在 attachBaseContext 中调用 this.getApplicationContext，也会返回空，因为 application 还没有创建完成。等到回调了 onCreate 时，Application 才算真正构造完毕。
