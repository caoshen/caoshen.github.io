---
title: ContentProvider 的 query 流程分析
date: 2019-09-11 00:34:13
tags: 四大组件
categories: Android
---

# ContentProvider 的 query 流程分析

ContentProvider 将底层的数据结构（比如数据库、文件）封装并且提供增删改查的接口，提供给本应用或者外部的应用调用。

## ContentResolver 的 query 方法

ContentProvider 的 query 操作是通过 ContentResolver 的 query 调用的，而不是直接调用 ContentProvider 的 query 方法，ContentProvider 的 query 方法类似于回调方法。

ContentResolver 的 query 方法一般调用方式如下：

context.getContentResolver().query(getUri(), null, null, null, null);

Context 是一个抽象类，它的具体实现都在 ContextImpl 类中。

ContextImpl 的 getContentResolver 方法如下：

```java
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }
```

mContentResolver 是 ContextImpl 构造时创建的。

构造如下：

```java
    private ContextImpl(@Nullable ContextImpl container, @NonNull ActivityThread mainThread,
            @NonNull LoadedApk packageInfo, @Nullable String splitName,
            @Nullable IBinder activityToken, @Nullable UserHandle user, int flags,
            @Nullable ClassLoader classLoader) {
        ...
        mContentResolver = new ApplicationContentResolver(this, mainThread);
    }
```

可以看出传入了 ContextImpl 自身和 mainThread（ActivityThread），构造出 ApplicationContentResolver。

ApplicationContentResolver 继承了 ContentResolver（ContentResolver是一个抽象类）。它的实现如下：

```java
    private static final class ApplicationContentResolver extends ContentResolver {
       ...
        @Override
        protected IContentProvider acquireProvider(Context context, String auth) {
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }
    }
```

可以看出它的 acquireProvider 方法直接调用了 ActivityThread 的 acquireProvider 方法，并返回一个 IContentProvider 对象。

ApplicationContentResolver 的 query 方法实际是它的父类 ContentResolver 的 query 方法。

ContentResolver 的 query 方法如下：

```java
    public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
            @Nullable String[] projection, @Nullable Bundle queryArgs,
            @Nullable CancellationSignal cancellationSignal) {
        ...
        IContentProvider unstableProvider = acquireUnstableProvider(uri);
        ...
        try {
            ...
            try {
                qCursor = unstableProvider.query(mPackageName, uri, projection,
                        queryArgs, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                // The remote process has died...  but we only hold an unstable
                // reference though, so we might recover!!!  Let's try!!!!
                // This is exciting!!1!!1!!!!1
                unstableProviderDied(unstableProvider);
                stableProvider = acquireProvider(uri);
                ...
                qCursor = stableProvider.query(
                        mPackageName, uri, projection, queryArgs, remoteCancellationSignal);
            }
            ...
    }
```

可以看出 ContentResolver 的 query 方法首先调用 acquireUnstableProvider 获取 unstableProvider。如果 unstableProvider 可以查询到数据，就使用 unstableProvider。否则就在 catch 中调用 acquireProvider 方法获取 stableProvider，使用 stableProvider 查询。不管是 unstableProvider 还是 stableProvider 的 query 方法，都是调用的 IContentProvider。

## IContentProvider

IContentProvider 代码如下：

```java
/**
 * The ipc interface to talk to a content provider.
 * @hide
 */
public interface IContentProvider extends IInterface {
    public Cursor query(String callingPkg, Uri url, @Nullable String[] projection,
            @Nullable Bundle queryArgs, @Nullable ICancellationSignal cancellationSignal)
            throws RemoteException;
    ...
    /* IPC constants */
    static final String descriptor = "android.content.IContentProvider";
}
```

可以看出 IContentProvider 与 AIDL 文件定义的 binder 接口类似，它也继承了 IInterface 接口。IContentProvider 的实现就是一个 binder 接口。

ActivityThread 的 acquireProvider 方法如下：

```java
    public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
            return provider;
        }
...    
            synchronized (getGetProviderLock(auth, userId)) {
                holder = ActivityManager.getService().getContentProvider(
                        getApplicationThread(), auth, userId, stable);
            }
        ...
        // Install provider will increment the reference count for us, and break
        // any ties in the race.
        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        return holder.provider;
    }
```

acquireProvider 方法主要有 3 步骤：

1. 获取已存在的 contentProvider（acquireExistingProvider）
2. 如果第 1 步获取不到，就向 AMS 获取 contentProvider（ActivityManager.getService().getContentProvider）
3. 安装第 2 步 的 contentprovider（installProvider）

acquireExistingProvider 方法的代码如下：

```java
    public final IContentProvider acquireExistingProvider(
            Context c, String auth, int userId, boolean stable) {
        synchronized (mProviderMap) {
            final ProviderKey key = new ProviderKey(auth, userId);
            final ProviderClientRecord pr = mProviderMap.get(key);
            if (pr == null) {
                return null;
            }
...
            IContentProvider provider = pr.mProvider;
            ... 
            return provider;
        }
    }
```

如果 mProviderMap 里面有要找的 providerClientRecord, 就返回 providerClientRecord 的 provider，否则返回 null。

mProviderMap 是一个 ProviderKey 和 ProviderClientRecord 的 ArrayMap。在 ActivityThread 的 installProviderAuthoritiesLocked 方法中会把 ProviderClientRecord 装入 mProviderMap。 

## AMS 获取 ContentProviderHolder

ActivityManager.getService().getContentProvider 方法调用了 AMS 的 getContentProvider 方法。最终会调到 AMS 的 getContentProviderImpl 方法。

getContentProviderImpl 首先会检查 ContentProvider 是否已经 publish 了。

```java
   private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
      ...
        ContentProviderRecord cpr;
      ...
            cpr = mProviderMap.getProviderByName(name, userId);
            ...
```

可以看出 getContentProviderImpl 会先从 mProviderMap 获取 ContentProviderRecord。

```java
if (providerRunning) {
                ...
                if (r != null && cpr.canRunHere(r)) {
                    // This provider has been published or is in the process
                    // of being published...  but it is also allowed to run
                    // in the caller's process, so don't make a connection
                    // and just let the caller instantiate its own instance.
                    ContentProviderHolder holder = cpr.newHolder(null);
                    // don't give caller the provider object, it needs
                    // to make its own.
                    holder.provider = null;
                    return holder;
                }
```

如果 cp 正在运行而且可以在调用者的进程中运行（canRunHere），那么直接返回这个 ContentProviderHolder。让调用者进程初始化 contentprovider。

```java
if (!providerRunning) {
               ...
                    cpi = AppGlobals.getPackageManager().
                        resolveContentProvider(name,
                            STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS, userId);
                 ...
                 ApplicationInfo ai =
                            AppGlobals.getPackageManager().
                                getApplicationInfo(
                                        cpi.applicationInfo.packageName,
                                        STOCK_PM_FLAGS, userId);
                       ...
                        ai = getAppInfoForUser(ai, userId);
                        cpr = new ContentProviderRecord(this, cpi, ai, comp, singleton);
```

如果 cp 没有在运行，从 packageManager 获取 ProviderInfo、ApplicationInfo，构造 ContentProviderRecord。

```java
if (proc != null && proc.thread != null && !proc.killed) {
                           ...
                            if (!proc.pubProviders.containsKey(cpi.name)) {
                                ...
                                    proc.thread.scheduleInstallProvider(cpi);
                               ...
                            }
                        } else {
                            ...
                            proc = startProcessLocked(cpi.processName,
                                    cpr.appInfo, false, 0, "content provider",
                                    new ComponentName(cpi.applicationInfo.packageName,
                                            cpi.name), false, false, false);
                            ...
                        }
```

如果 cp 的进程启动了，但是 cp 没有 publish，需要通知应用安装 provider（ scheduleInstallProvider）。

如果 cp 的进程没有启动，启动进程（startProcessLocked）。

```java
// Wait for the provider to be published...
        synchronized (cpr) {
            while (cpr.provider == null) {
                ...
                try {
                    ...
                    if (conn != null) {
                        conn.waiting = true;
                    }
                    cpr.wait();
                } catch (InterruptedException ex) {
                } finally {
                    if (conn != null) {
                        conn.waiting = false;
                    }
                    ...
                }
            }
        }
```

getContentProviderImpl 的最后有一个无限循环，一直轮询等待，直到 contentProviderRecord的 provider 不为空。

AMS 的 getContentProviderImpl 这部分主要是为了获取 ContentProviderHolder，如果能从缓存中读取到，就出缓存中取，否则创建新的 ContentProviderHolder。

## ApplicationThread 的 scheduleInstallProvider

```java
        @Override
        public void scheduleInstallProvider(ProviderInfo provider) {
            sendMessage(H.INSTALL_PROVIDER, provider);
        }
```

第三步可以看到调用了 proc.thread.scheduleInstallProvider(cpi) 安装 provider。它的实际操作是发消息给 ActivityThread，让它 installProvider。

ActivityThread 的 installContentProviders 方法如下：

```java
    private void installContentProviders(
            Context context, List<ProviderInfo> providers) {
        final ArrayList<ContentProviderHolder> results = new ArrayList<>();

        for (ProviderInfo cpi : providers) {
            ...
            ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            ...
        }

        ...
            ActivityManager.getService().publishContentProviders(
                getApplicationThread(), results);
        ...
    }
```

可以看出 installContentProviders 做了 2 件事。

1. installProvider;
2. 通知 AMS publishContentProviders

ActivityThread 的 installProvider 方法如下：

```java
    private ContentProviderHolder installProvider(Context context,
            ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ...
                localProvider = packageInfo.getAppFactory()
                        .instantiateProvider(cl, info.name);
                provider = localProvider.getIContentProvider();
                ...
        } else {
            provider = holder.provider;
           ...
        }

        ContentProviderHolder retHolder;

        synchronized (mProviderMap) {
            if (DEBUG_PROVIDER) Slog.v(TAG, "Checking to add " + provider
                    + " / " + info.name);
            IBinder jBinder = provider.asBinder();
            if (localProvider != null) {
                ...
                retHolder = pr.mHolder;
            } else {
                ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
                ...
                    mProviderRefCountMap.put(jBinder, prc);
                ...
                retHolder = prc.holder;
            }
        }
        return retHolder;
    }
```

可以看出如果 holder 的 provider 为空，就说明是本地 ContentProvider，这时创建本地的 ContentProvider，然后调用它的 getIContentProvider方法。
如果 holder 的 provider 不为空，说明是外部 ContentProvider，就给 mProviderRefCountMap 增加引用计数。

## Transport 的 query

获取到 ContentProviderHolder 后，实际获取的是 ContentProvider 的 mTransport 对象，调用的是 mTransport 的 query 方法。

```java
    public IContentProvider getIContentProvider() {
        return mTransport;
    }
```

mTransport 是 Transport 类型，它是 ContentProvider 的内部类。

```java
    class Transport extends ContentProviderNative {
```

ContentProviderNative 是实现了 getIContentProvider 接口的 Binder 类，因此它可以跨进程调用。

Transport 的 query 方法如下：

```java
        public Cursor query(String callingPkg, Uri uri, @Nullable String[] projection,
                @Nullable Bundle queryArgs, @Nullable ICancellationSignal cancellationSignal) {
            validateIncomingUri(uri);
            uri = maybeGetUriWithoutUserId(uri);
            ...
                return ContentProvider.this.query(
                        uri, projection, queryArgs,
                        CancellationSignal.fromTransport(cancellationSignal));
            ...
        }
```

可以看出 Transport 调用了它的父类 ContentProvider 的 query 方法。因此最后可以执行到 ContentProvider 的 query，完成查询操作。