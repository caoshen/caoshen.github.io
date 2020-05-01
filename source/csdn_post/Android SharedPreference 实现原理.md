# Android SharedPreferences 实现原理

Android 的 SharedPreferences 一般用来保存一些标记位或者是一些设置项，比较适合轻量级的存储。

通常获取 SharedPreferences 时有两种方法：

1. 通过 PreferenceManager 的 getDefaultSharedPreferences 得到默认的 SharedPreferences，也就是应用包名开头的 SharedPreferences。

```java
SharedPreferences sharedPreferences = PreferenceManager.getDefaultSharedPreferences(context);
```

2. 通过 Context 的 getSharedPreferences 方法获取指定名称的 SharedPreferences。

```java
context.getSharedPreferences("sp_name", Context.MODE_PRIVATE);
```

## SharedPreference 的读写模式

SharedPreferences 是保存在 data/data/包名 目录下的 xml 文件，每个配置项都以键值对的形式保存在 xml 中。

getSharedPreferences 的第二个参数表示文件的读写模式，除了私有读写模式（MODE_PRIVATE），其他 3 种模式都已经被标记为过时（@Deprecated）。

这 4 种模式声明如下：

```java
    public static final int MODE_PRIVATE = 0x0000;

    @Deprecated
    public static final int MODE_WORLD_READABLE = 0x0001;

    @Deprecated
    public static final int MODE_WORLD_WRITEABLE = 0x0002;

    /**...
     * @deprecated MODE_MULTI_PROCESS does not work reliably in
     * some versions of Android, and furthermore does not provide any
     * mechanism for reconciling concurrent modifications across
     * processes.  Applications should not attempt to use it.  Instead,
     * they should use an explicit cross-process data management
     * approach such as {@link android.content.ContentProvider ContentProvider}.
     */
    @Deprecated
    public static final int MODE_MULTI_PROCESS = 0x0004;
```

MODE_PRIVATE 表示只有本应用可以读写。

MODE_WORLD_READABLE 和 MODE_WORLD_WRITEABLE 表示其他应用也可以读写，这 2 种会导致安全问题，其他应用可以修改和攻击本应用的 SharedPreferences 值。

MODE_MULTI_PROCESS 表示支持多进程访问，每次 SharedPreferences 被修改，data/data 目录下的 xml 文件都会重新加载文件内容到内存中。但是在一些 Android 版本上无法保证多进程并发读写的可靠性，因此更加推荐使用 ContentProvider 进行跨进程读写。

## SharedPreferences 和 SharedPreferencesImpl

SharedPreference 是一个接口，接口提供了基本类型和 String 的读写操作，比如 putString、getString 等。

```java
public interface SharedPreferences {
```

SharedPreference 也提供了 OnSharedPreferenceChangeListener 接口，用来监听 SharedPreferences 的变化。需要注意的是 OnSharedPreferenceChangeListener 接口回调在主线程。

```java
    public interface OnSharedPreferenceChangeListener {
        void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key);
    }
```

SharedPreferences 的实现都在 SharedPreferencesImpl 中，这有点类似 Context 和 ContextImpl，但是 Context 是抽象类，而 SharedPreferences 是接口。

回到 Context 的 getSharedPreferences 方法，它的真正实现在 ContextImpl 的 getSharedPreferences 方法中。

```java
    @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        ...
        File file;
        synchronized (ContextImpl.class) {
            if (mSharedPrefsPaths == null) {
                mSharedPrefsPaths = new ArrayMap<>();
            }
            file = mSharedPrefsPaths.get(name);
            if (file == null) {
                file = getSharedPreferencesPath(name);
                mSharedPrefsPaths.put(name, file);
            }
        }
        return getSharedPreferences(file, mode);
    }
```

可以看出 getSharedPreferences 会先根据 SharedPreferences 的名称找到 SharedPreferences 对应的文件，然后调用 getSharedPreferences(file, mode) 方法。

```java
    @Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
            if (sp == null) {
                checkMode(mode);
                if (getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.O) {
                    if (isCredentialProtectedStorage()
                            && !getSystemService(UserManager.class)
                                    .isUserUnlockingOrUnlocked(UserHandle.myUserId())) {
                        throw new IllegalStateException("SharedPreferences in credential encrypted "
                                + "storage are not available until after user is unlocked");
                    }
                }
                sp = new SharedPreferencesImpl(file, mode);
                cache.put(file, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }
```

可以看出 getSharedPreferences(file, mode) 方法中使用到了 ArrayMap<File, SharedPreferencesImpl> 类型的缓存，用来保存所有的 SharedPreferences。如果缓存中的 SharedPreferences 为空，就先检查权限，然后构造一个新的 SharedPreferencesImpl 存入缓存中并返回。如果是 MODE_MULTI_PROCESS 多进程模式或者目标版本低于 3.0，会重新加载 xml 文件到内存中。

查看 SharedPreferencesImpl 的构造方法，可以发现它调用了 startLoadFromDisk 用来加载 xml。

```java
    SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        mThrowable = null;
        startLoadFromDisk();
    }
```

startLoadFromDisk 方法启动了一个线程，在线程中调用了 loadFromDisk 方法。

loadFromDisk 方法如下：

```java
    private void loadFromDisk() {
        ...
                try {
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16 * 1024);
                    map = (Map<String, Object>) XmlUtils.readMapXml(str);
                } catch (Exception e) {
        ...
                    if (map != null) {
                        mMap = map;
        ...
    }
```

可以看出 SharedPreferencesImpl 里面的 Map<String, Object> mMap 键值对就是从 xml 里面读取出来的。

SharedPreferencesImpl 的读操作也是直接从 mMap 缓存中读取。

SharedPreferencesImpl 的 getString 方法如下：

```java
    @Override
    @Nullable
    public String getString(String key, @Nullable String defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            String v = (String)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
```

可以看出 getString 是线程安全的，它用 synchronized 同步代码块保护着。读取时直接从 mMap get，如果 get 不到就返回默认值。

## commit 和 apply

修改 SharedPreferences 的值之后一般会使用 commit 或者 apply 保存修改的值到内存和文件中。

```java
        ...
        boolean commit();
        ...
        void apply();
```

commit 和 apply 主要有 2 点区别。

1. commit 是同步操作，apply 是异步操作
2. commit 有返回值，apply 没有返回值

如果不在乎是否操作成功的返回值，或者不想在当前线程直接写文件，那么建议使用 apply 代替 commit。

如果是 commit 方法，会直接调用 writeToFile 将内存中的 map 写入 xml 文件。

```java
    @GuardedBy("mWritingToDiskLock")
    private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
```

如果是 apply 方法，会封装一个 writeToDiskRunnable，并将它加入到任务队列中，由一个 HandlerThread 异步执行。

入队方法如下：

```java
    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final boolean isFromSyncCommit = (postWriteRunnable == null);
...
        QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
    }
```

不管是 commit 或者 apply，写入后都会调用 notifyListeners 通知回调接口。

```java
        private void notifyListeners(final MemoryCommitResult mcr) {
            ...
            if (Looper.myLooper() == Looper.getMainLooper()) {
                for (int i = mcr.keysModified.size() - 1; i >= 0; i--) {
                    final String key = mcr.keysModified.get(i);
                    for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
                        if (listener != null) {
                            listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, key);
                        }
                    }
                }
            } else {
                // Run this function on the main thread.
                ActivityThread.sMainThreadHandler.post(() -> notifyListeners(mcr));
            }
        }
    }
```

可以看出 OnSharedPreferenceChangeListener 接口都会在主线程执行。如果本来就是主线程，直接调用回调接口。否则 post 到主线程再回调。

## SharedPreferences 跨进程

从 getSharedPreferences 的实现可以看出，如果模式是 MODE_MULTI_PROCESS 多进程模式，或者 targetSdkVersion 目标版本低于 3.0，那么每次读写 SharedPreferences 都会重新加载一遍 xml 文件。

```java
    @Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        ...
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }
```

SharedPreferencesImpl 的 startReloadIfChangedUnexpectedly 方法如下：

```java
    void startReloadIfChangedUnexpectedly() {
        synchronized (mLock) {
            // TODO: wait for any pending writes to disk?
            if (!hasFileChangedUnexpectedly()) {
                return;
            }
            startLoadFromDisk();
        }
    }
```

可以看出如果 SharedPreferences 的 xml 文件没有被改动过，直接返回。否则从磁盘上重新加载 xml 文件到内存。但是就像注释中的 TODO 说的，它没有等待任何还没有执行完的写操作先写入磁盘完毕，也就是说读取的 SharedPreferences 可能不是最新的，其他进程异步的写操作可能还没有写入 xml 文件。因此 MODE_MULTI_PROCESS 多进程模式也无法保证多进程并发读写的可靠性，被标记为 @deprecated。

