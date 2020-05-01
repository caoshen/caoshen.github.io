# VirtualAPK 插件化原理分析

这里以 VirtualAPK 0.9.8 版本为例，从三个方面介绍 VirtualAPK 的插件化原理。

1. VirtualAPK 如何加载插件 APK 的类
2. VirtualAPK 如何解决宿主 APK 和插件 APK 的冲突
3. VirtualAPK 如何支持四大组件

## VirtualAPK 插件类的加载

VirtualAPK 从外部存储读取到插件 APK 后，调用了 PluginManager 的 loadPlugin 的方法加载插件 APK。

```java
        File plugin = new File(pluginPath);
        PluginManager.getInstance(this).loadPlugin(plugin);
```

PluginManager 的 loadPlugin 方法如下：

```java
    public void loadPlugin(File apk) throws Exception {
...
        LoadedPlugin plugin = createLoadedPlugin(apk);
        
        if (null == plugin) {
            throw new RuntimeException("Can't load plugin which is invalid: " + apk.getAbsolutePath());
        }
...
    }
```

可以看出loadPlugin 会调用 createLoadedPlugin 方法得到一个 LoadedPlugin。

```java
    protected LoadedPlugin createLoadedPlugin(File apk) throws Exception {
        return new LoadedPlugin(this, this.mContext, apk);
    }
```

可以看出 createLoadedPlugin 调用了 LoadedPlugin 的构造方法，将 PluginManager、context、插件 APK 作为参数传递给了构造方法。

LoadedPlugin 的构造方法如下：

```java
    public LoadedPlugin(PluginManager pluginManager, Context context, File apk) throws Exception {
        this.mPluginManager = pluginManager;
        ...
        this.mPackage = PackageParserCompat.parsePackage(context, apk, PackageParser.PARSE_MUST_BE_APK);
        ...
        this.mResources = createResources(context, getPackageName(), apk);
        this.mClassLoader = createClassLoader(context, apk, this.mNativeLibDir, context.getClassLoader());
...
        // Cache instrumentations
        Map<ComponentName, InstrumentationInfo> instrumentations = new HashMap<ComponentName, InstrumentationInfo>();
...
        // Cache activities
        Map<ComponentName, ActivityInfo> activityInfos = new HashMap<ComponentName, ActivityInfo>();
...

        // Cache services
        Map<ComponentName, ServiceInfo> serviceInfos = new HashMap<ComponentName, ServiceInfo>();
...

        // Cache providers
        Map<String, ProviderInfo> providers = new HashMap<String, ProviderInfo>();
...

        // Register broadcast receivers dynamically
        Map<ComponentName, ActivityInfo> receivers = new HashMap<ComponentName, ActivityInfo>();
...
        // try to invoke plugin's application
        invokeApplication();
    }
```

可以看出 LoadedPlugin 的构造方法会解析 APK 包，加载插件资源，加载插件类，解析 Instrumentation 和四大组件，最后调用插件的 application。

使用 PackageParserCompat.parsePackage 解析 APK 包。

```java
        this.mPackage = PackageParserCompat.parsePackage(context, apk, PackageParser.PARSE_MUST_BE_APK);
```

使用 createResources 加载插件资源。

```java
        this.mResources = createResources(context, getPackageName(), apk);
```

使用 createClassLoader 加载插件类。

```java
        this.mClassLoader = createClassLoader(context, apk, this.mNativeLibDir, context.getClassLoader());
```

从 PackageParser.Package 解析 Instrumentation 和四大组件。

```java
        // Cache activities
        Map<ComponentName, ActivityInfo> activityInfos = new HashMap<ComponentName, ActivityInfo>();
        for (PackageParser.Activity activity : this.mPackage.activities) {
            activity.info.metaData = activity.metaData;
            activityInfos.put(activity.getComponentName(), activity.info);
        }
        this.mActivityInfos = Collections.unmodifiableMap(activityInfos);
        this.mPackageInfo.activities = activityInfos.values().toArray(new ActivityInfo[activityInfos.size()]);
```

最后调用了 invokeApplication 方法运行插件 APK 的 application。

插件类的加载 createClassLoader 方法如下：

```java
    protected ClassLoader createClassLoader(Context context, File apk, File libsDir, ClassLoader parent) throws Exception {
        File dexOutputDir = getDir(context, Constants.OPTIMIZE_DIR);
        String dexOutputPath = dexOutputDir.getAbsolutePath();
        DexClassLoader loader = new DexClassLoader(apk.getAbsolutePath(), dexOutputPath, libsDir.getAbsolutePath(), parent);

        if (Constants.COMBINE_CLASSLOADER) {
            DexUtil.insertDex(loader, parent, libsDir);
        }

        return loader;
    }
```

可以看出 createClassLoader 构造了 DexCloader 并且把宿主 APK 的 classLoader 传递给了插件 APK 的 DexClassLoader。因此 VirtualAPK 的插件可以调用宿主的类。

```java
        DexClassLoader loader = new DexClassLoader(apk.getAbsolutePath(), dexOutputPath, libsDir.getAbsolutePath(), parent);
```

在 createClassLoader 方法里面也判断了 Constants.COMBINE_CLASSLOADER 是否为 true，如果为 true 就调用 insertDex 方法。

insertDex 方法把插件 APK 的 Dex 和宿主 APK 的 Dex 合并，然后把插件 APK 的 Dex 排在宿主 APK 后面，这样就在宿主 APK 包含了所有的 Dex，宿主可以加载插件 APK 的类。

合并宿主 APK 和 插件 APK 的 dex 的方法如下：

```java
    public static void insertDex(DexClassLoader dexClassLoader, ClassLoader baseClassLoader, File nativeLibsDir) throws Exception {
        Object baseDexElements = getDexElements(getPathList(baseClassLoader));
        Object newDexElements = getDexElements(getPathList(dexClassLoader));
        Object allDexElements = combineArray(baseDexElements, newDexElements);
        Object pathList = getPathList(baseClassLoader);
        Reflector.with(pathList).field("dexElements").set(allDexElements);

        insertNativeLibrary(dexClassLoader, baseClassLoader, nativeLibsDir);
    }
```

注意到 Constants.COMBINE_CLASSLOADER 这个常量应该始终为 true。否则宿主 APK 就不会包含插件 APK 的 dex，也就加载不了插件的类。

## VirtualAPK 资源加载和冲突

### 资源加载

LoadedPlugin 类的构造方法调用了 createResources 方法来构造资源。

```java
        this.mResources = createResources(context, getPackageName(), apk);
```

createResources 方法如下：

```java
    protected Resources createResources(Context context, String packageName, File apk) throws Exception {
        if (Constants.COMBINE_RESOURCES) {
            return ResourcesManager.createResources(context, packageName, apk);
        } else {
            Resources hostResources = context.getResources();
            AssetManager assetManager = createAssetManager(context, apk);
            return new Resources(assetManager, hostResources.getDisplayMetrics(), hostResources.getConfiguration());
        }
    }
```

注意到如果 Constants.COMBINE_RESOURCES 为 true，那么 createResources 会合并宿主 APK 和插件 APK 的资源，否则只返回插件 APK 的资源。

ResourcesManager 的 createResources 方法如下：

```java
    public static synchronized Resources createResources(Context hostContext, String packageName, File apk) throws Exception {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
            return createResourcesForN(hostContext, packageName, apk);
        }
        
        Resources resources = ResourcesManager.createResourcesSimple(hostContext, apk.getAbsolutePath());
        ResourcesManager.hookResources(hostContext, resources);
        return resources;
    }
```

如果是 N 版本以上 SDK，会调用 createResourcesForN 方法。

```java
    /**
     * Use System Apis to update all existing resources.
     * <br/>
     * 1. Update ApplicationInfo.splitSourceDirs and LoadedApk.mSplitResDirs
     * <br/>
     * 2. Replace all keys of ResourcesManager.mResourceImpls to new ResourcesKey
     * <br/>
     * 3. Use ResourcesManager.appendLibAssetForMainAssetPath(appInfo.publicSourceDir, "${packageName}.vastub") to update all existing resources.
     * <br/>
     *
     * see android.webkit.WebViewDelegate.addWebViewAssetPath(Context)
     */
    @TargetApi(Build.VERSION_CODES.N)
    private static Resources createResourcesForN(Context context, String packageName, File apk) throws Exception {
...
        LoadedApk loadedApk = Reflector.with(context).field("mPackageInfo").get();
    
        Reflector rLoadedApk = Reflector.with(loadedApk).field("mSplitResDirs");
        String[] splitResDirs = rLoadedApk.get();
        rLoadedApk.set(append(splitResDirs, newAssetPath));
...
        // lastly, sync all LoadedPlugin to newResources
        for (LoadedPlugin plugin : PluginManager.getInstance(context).getAllLoadedPlugins()) {
            plugin.updateResources(newResources);
        }
...
        return newResources;
    }
```

通过方法注释可以看出 createResourcesForN 方法更新了宿主 ApplicationInfo 的 splitSourceDirs、宿主 LoadedApk 的 mSplitResDirs、ResourcesManager.mResourceImpls 的 key 值、ResourcesManager 的 assetPath，最后把得到的新 Resources 更新给了每个插件的 LoadedPlugin。所以宿主 APK 和 插件 APK 可以相互共享资源。

### 资源冲突

如果插件资源和宿主资源重复，比如引用了同一个 support 包，那么 virtualAPK 会过滤掉插件 APK 引用的 support 包。

使用 virtualAPK 是有 2 个有关资源的问题：

1. 如果插件依赖宿主 1.0 版本编译，宿主出了 2.0 版本，插件找不到 1.0 的资源 id 怎么办？ 
2. 怎么解决插件和宿主资源同名但是不同内容的问题？比如两者都有一个叫做 about 的字符串，关于的内容不一样。

对于第一个问题，如果宿主出了 2.0 版本，那么插件也会出 2.0 版本，保持一致。

对于第二个问题，同名的资源在构建的时候 gradle 会报错，提示 duplicate resources entry，从 Android 开发的角度来看，资源不能重复，否则 merge 的时候就会报错。因此需要保证资源不同名。

## VirtualAPK 四大组件的支持

### Activity 的支持

Activity 的支持可以使用“欺上瞒下”来概括。

“欺上”是指启动一个 StubActivity 来欺骗 AMS，让 AMS 认为 Activity 已经在 AndroidManifest 注册，但是实际注册的 VirtualAPK 的 core aar 里面的 AndroidManifest 的占坑 Activity，比如

```
        <!-- Stub Activities -->
        <activity
            android:name="com.didi.virtualapk.core.A$1"
            android:exported="false"
            android:launchMode="standard" />
```

“瞒下”是指 ActivityThread 在 performLaunchActivity 方法中启动 Activity 会调用 Instrumentation 的 newActivity 创建 Activity，此时创建的不是占坑的 StubActivity，而是真正需要启动的插件 Activity。

VirtualAPK 的“欺上瞒下”是通过 hook 替换原始 Instrumentation，改为自己的 VAInstrumentation。

VAInstrumentation 构造方法如下：

```java
public class VAInstrumentation extends Instrumentation implements Handler.Callback {
    ...
    public VAInstrumentation(PluginManager pluginManager, Instrumentation base) {
        this.mPluginManager = pluginManager;
        this.mBase = base;
    }
```

可以看出真正的 Instrumentation 作为了 mBase 变量，有点类似 ContextImpl。

VAInstrumentation 主要做 2 件事：

1. 启动时替换原始 intent 为占坑 intent
2. 构造 Activity 时还原占坑 intent 为原始 intent

启动时替换原始 intent 为占坑 intent，这一步依靠 injectIntent 来完成，注入 Intent 后还是调用 Instrumentation 类启动 Intent。

```java
    @Override
    public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode) {
        injectIntent(intent);
        return mBase.execStartActivity(who, contextThread, token, target, intent, requestCode);
    }
```

injectIntent 方法如下：

```java
    protected void injectIntent(Intent intent) {
        ...
        // null component is an implicitly intent
        if (intent.getComponent() != null) {
            ...
            // resolve intent with Stub Activity if needed
            this.mPluginManager.getComponentsHandler().markIntentIfNeeded(intent);
        }
    }
```

可以看出调用了 markIntentIfNeeded 来转换为 StubActivity。

markIntentIfNeeded 方法如下：

```java
    public void markIntentIfNeeded(Intent intent) {
        if (intent.getComponent() == null) {
            return;
        }

        String targetPackageName = intent.getComponent().getPackageName();
        String targetClassName = intent.getComponent().getClassName();
        // search map and return specific launchmode stub activity
        if (!targetPackageName.equals(mContext.getPackageName()) && mPluginManager.getLoadedPlugin(targetPackageName) != null) {
            intent.putExtra(Constants.KEY_IS_PLUGIN, true);
            intent.putExtra(Constants.KEY_TARGET_PACKAGE, targetPackageName);
            intent.putExtra(Constants.KEY_TARGET_ACTIVITY, targetClassName);
            dispatchStubActivity(intent);
        }
    }
```

markIntentIfNeeded 会把原始 intent 的包名和类名存入 intent 的 extra 中，然后调用 dispatchStubActivity 启动 StubActivity。

dispatchStubActivity 方法如下：

```java
    private void dispatchStubActivity(Intent intent) {
        ComponentName component = intent.getComponent();
        String targetClassName = intent.getComponent().getClassName();
        ...
        String stubActivity = mStubActivityInfo.getStubActivity(targetClassName, launchMode, themeObj);
        ...
        intent.setClassName(mContext, stubActivity);
    }
```

可以看出 dispatchStubActivity 根据类名、启动模式、主题来选择合适的占坑 Activity，最后用 setClassName 改写原始 Activity 为占坑 Activity。

VAInstrumentation 的 newActivity 用来构造 Activity ，还原占坑 intent 为原始 intent。

```java
    @Override
    public Activity newActivity(ClassLoader cl, String className, Intent intent) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
        try {
            cl.loadClass(className);
            ...
        } catch (ClassNotFoundException e) {
            ComponentName component = PluginUtil.getComponent(intent);
            ...
            Activity activity = mBase.newActivity(plugin.getClassLoader(), targetClassName, intent);
            ...
            return newActivity(activity);
        }

        return newActivity(mBase.newActivity(cl, className, intent));
    }
```

newActivity 中先判断用宿主的 classLoader 能否加载启动的类，如果能加载说明时宿主的 Activity，直接启动。否则调用 PluginUtil.getComponent 得到原始的插件 Intent，然后使用插件的 ClassLoader 来加载原始的插件 Intent。

### Service 的支持

Service 的支持也是采用“欺上瞒下”的方法，插件 Service 的启动实际上是通过 VirtualAPK 的代理 Service 启动的。

正常的 Service 启动会调用 ContextImpl 的 startServiceCommon 方法。

startServiceCommon 方法如下：

```java
    private ComponentName startServiceCommon(Intent service, boolean requireForeground,
            UserHandle user) {
        try {
            ...
            ComponentName cn = ActivityManager.getService().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), requireForeground,
                            getOpPackageName(), user.getIdentifier());
            ...
    }
```

可以看出 startServiceCommon 方法调用了 AMS 的 startService。

为了调用 VirtualAPK 的代理 Service，VirtualAPK 的 pluginManager hook 替换了 ActivityManager 为自定义的 ActivityManagerProxy。采用动态代理的方式在 AMS startService 之前修改了 intent。

PluginManager 的构造方法会调用 hookCurrentProcess 方法。

```java
    protected PluginManager(Context context) {
        ...
        hookCurrentProcess();
    }

    protected void hookCurrentProcess() {
        hookInstrumentationAndHandler();
        hookSystemServices();
        hookDataBindingUtil();
    }
```

hookCurrentProcess 的第二步就是 hookSystemServices，它主要 hook 了 ActivityManager。


```java
    protected void hookSystemServices() {
            ...
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                defaultSingleton = Reflector.on(ActivityManager.class).field("IActivityManagerSingleton").get();
            }
            ...
            IActivityManager origin = defaultSingleton.get();
            IActivityManager activityManagerProxy = (IActivityManager) Proxy.newProxyInstance(mContext.getClassLoader(), new Class[] { IActivityManager.class },
                createActivityManagerProxy(origin));

            // Hook IActivityManager from ActivityManagerNative
            Reflector.with(defaultSingleton).field("mInstance").set(activityManagerProxy);
            ...
    }
```

可以看出使用 Proxy.newProxyInstance 生成了一个动态代理的 IActivityManager 对象。这样在调用 IActivityManager 的原始方法时，会转而调用 activityManagerProxy 代理的 invoke 方法。

createActivityManagerProxy 方法如下：

```java
    protected ActivityManagerProxy createActivityManagerProxy(IActivityManager origin) throws Exception {
        return new ActivityManagerProxy(this, origin);
    }
```

createActivityManagerProxy 构造了一个 ActivityManagerProxy 对象，并传递原始的 ActivityManager。

```java
public class ActivityManagerProxy implements InvocationHandler {
...
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if ("startService".equals(method.getName())) {
            try {
                return startService(proxy, method, args);
            } catch (Throwable e) {
                Log.e(TAG, "Start service error", e);
            }
        }
        ...
    }
```

可以看出 startService 方法会调用 ActivityManagerProxy 的 startService。

```java
    protected Object startService(Object proxy, Method method, Object[] args) throws Throwable {
        ...
        return startDelegateServiceForTarget(target, resolveInfo.serviceInfo, null, RemoteService.EXTRA_COMMAND_START_SERVICE);
    }
```

startService 方法会先判断启动的是宿主的 Service 还是插件的 Service。如果是插件的 Service，直接调用 startDelegateServiceForTarget。startDelegateServiceForTarget 会启动代理 Service。

```java
    protected ComponentName startDelegateServiceForTarget(Intent target, ServiceInfo serviceInfo, Bundle extras, int command) {
        Intent wrapperIntent = wrapperTargetIntent(target, serviceInfo, extras, command);
        return mPluginManager.getHostContext().startService(wrapperIntent);
    }

    protected Intent wrapperTargetIntent(Intent target, ServiceInfo serviceInfo, Bundle extras, int command) {
        // fill in service with ComponentName
        target.setComponent(new ComponentName(serviceInfo.packageName, serviceInfo.name));
        String pluginLocation = mPluginManager.getLoadedPlugin(target.getComponent()).getLocation();

        // start delegate service to run plugin service inside
        boolean local = PluginUtil.isLocalService(serviceInfo);
        Class<? extends Service> delegate = local ? LocalService.class : RemoteService.class;
        Intent intent = new Intent();
        intent.setClass(mPluginManager.getHostContext(), delegate);
        intent.putExtra(RemoteService.EXTRA_TARGET, target);
        intent.putExtra(RemoteService.EXTRA_COMMAND, command);
        intent.putExtra(RemoteService.EXTRA_PLUGIN_LOCATION, pluginLocation);
        if (extras != null) {
            intent.putExtras(extras);
        }

        return intent;
    }
```

可以看出 startDelegateServiceForTarget 修改了 intent，它将原始 intent 作为参数直接传递给了新 intent。注意到 intent 本身实现了 Parcelable 接口，它可以作为参数传递。

如果是本地 Service，就启动 LocalService 作为代理。如果是多进程 Service，就启动 RemoteService 作为代理。

查看 VirtualAPK 的 manifest。

```
        <!-- Local Service running in main process -->
        <service
            android:name="com.didi.virtualapk.delegate.LocalService"
            android:exported="false" />

        <!-- Daemon Service running in child process -->
        <service
            android:name="com.didi.virtualapk.delegate.RemoteService"
            android:exported="false"
            android:process=":daemon" >
            <intent-filter>
                <action android:name="${applicationId}.intent.ACTION_DAEMON_SERVICE" />
            </intent-filter>
        </service>
```

可以看出 LocalService 运行在主进程，RemoteService 运行在 daemon 新进程。

LocalService 的 onStartCommand 代码如下：

```java
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ...
        int command = intent.getIntExtra(EXTRA_COMMAND, 0);
        ...
        switch (command) {
            case EXTRA_COMMAND_START_SERVICE: {
                ...
                    try {
                        service = (Service) plugin.getClassLoader().loadClass(component.getClassName()).newInstance();
...
                        attach.invoke(service, plugin.getPluginContext(), mainThread, component.getClassName(), token, app, am);
                        service.onCreate();
                        this.mPluginManager.getComponentsHandler().rememberService(component, service);
                    } catch (Throwable t) {
                        return START_STICKY;
                    }
                }

                service.onStartCommand(target, 0, this.mPluginManager.getComponentsHandler().getServiceCounter(service).getAndIncrement());
                break;
            }
            ...
        return START_STICKY;
    }
```

可以看出 LocalService 在 onStartCommand 时会从 intent 取出 command，代表需要执行哪些操作。比如 startService 会调用 classloader 加载 Service 的 class，然后依次调用 service 的 attach、onCreate、onStartCommand 方法，完成插件 Service 的启动。

## ContentProvider 的支持

和 StubActivity、LocalService 一样，VirtualAPK 在 AndroidManifest 提供了一个代理 ContentProvider，即 RemoteContentProvider。

```
        <provider
            android:name="com.didi.virtualapk.delegate.RemoteContentProvider"
            android:authorities="${applicationId}.VirtualAPK.Provider"
            android:exported="false"
            android:process=":daemon" />
```

正常流程下通过 getContentResolver 查询 ContentProvider 时，实际上调用了 ContextImpl 的 ApplicationContentResolver 去查询。

VirtualAPK 在加载插件时，会产生一个 LoadedPlugin，并且为 LoadedPlugin 创建 pluginContext，pluginContext 的 getContentResolver 方法会构造一个 PluginContentResolver。

```java
    @Override
    public ContentResolver getContentResolver() {
        return new PluginContentResolver(getHostContext());
    }
```

正常流程下 ApplicationContentResolver 查询时会调用 ActivityThread 的 acquireProvider 方法。

在插件的 PluginContentResolver 中会调用 PluginManager 的 getIContentProvider 方法。

```java
    @Override
    protected IContentProvider acquireProvider(Context context, String auth) {
        if (mPluginManager.resolveContentProvider(auth, 0) != null) {
            return mPluginManager.getIContentProvider();
        }
        return super.acquireProvider(context, auth);
    }
```

PluginManager 的 getIContentProvider 方法调用了 hookIContentProviderAsNeeded。 hookIContentProviderAsNeeded 会 hook ActivityThread 中的 IContentProvider.

```java
    public synchronized IContentProvider getIContentProvider() {
        if (mIContentProvider == null) {
            hookIContentProviderAsNeeded();
        }

        return mIContentProvider;
    }
```
hookIContentProviderAsNeeded 方法如下：

```java
    protected void hookIContentProviderAsNeeded() {
        Uri uri = Uri.parse(RemoteContentProvider.getUri(mContext));
        mContext.getContentResolver().call(uri, "wakeup", null, null);
        try {
            ...
            ActivityThread activityThread = ActivityThread.currentActivityThread();
            Map providerMap = Reflector.with(activityThread).field("mProviderMap").get();
            Iterator iter = providerMap.entrySet().iterator();
            while (iter.hasNext()) {
                ...
                if (auth.equals(RemoteContentProvider.getAuthority(mContext))) {
                    ...
                    IContentProvider rawProvider = (IContentProvider) provider.get(val);
                    IContentProvider proxy = IContentProviderProxy.newInstance(mContext, rawProvider);
                    mIContentProvider = proxy;
                    Log.d(TAG, "hookIContentProvider succeed : " + mIContentProvider);
                    break;
                }
            }
        } catch (Exception e) {
            Log.w(TAG, e);
        }
    }
```

可以看出 hookIContentProviderAsNeeded 首先解析得到占坑的代理 Provider，即 RemoteContentProvider 的 Uri，然后立即 call 了占坑的 RemoteContentProvider。接着从 ActivityThread 的 mProviderMap 中查找对应 auth 的 provider。如果存在和 RemoteContentProvider 相同 auth 的 provider，就采用动态代理的方式 hook 掉这个 provider，替换为自己的 IContentProviderProxy 类。

IContentProviderProxy 是一个代理接口，它实现了 InvocationHandler，直接看它的 invoke 方法。

```java
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Log.v(TAG, method.toGenericString() + " : " + Arrays.toString(args));
        wrapperUri(method, args);

        try {
            return method.invoke(mBase, args);
        } catch (InvocationTargetException e) {
            throw e.getTargetException();
        }
    }
```

可以看出 invoke 方法封装了 method、args，然后调用原始 IContentProvider 的方法。

wrapperUri 会调用 PluginContentResolver 的 wrapperUri 方法。

```java
    @Keep
    public static Uri wrapperUri(LoadedPlugin loadedPlugin, Uri pluginUri) {
        String pkg = loadedPlugin.getPackageName();
        String pluginUriString = Uri.encode(pluginUri.toString());
        StringBuilder builder = new StringBuilder(RemoteContentProvider.getUri(loadedPlugin.getHostContext()));
        builder.append("/?plugin=" + loadedPlugin.getLocation());
        builder.append("&pkg=" + pkg);
        builder.append("&uri=" + pluginUriString);
        Uri wrapperUri = Uri.parse(builder.toString());
        return wrapperUri;
    }
```

wrapperUri 将插件的存放位置、包名、uri 三者作为参数成为新的 uri。它的作用是在代理 RemoteContentProvider 分发增删改查操作时，再把这些信息提取出来。

这里以 RemoteContentProvider 的 query 方法为例。

```java
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        ContentProvider provider = getContentProvider(uri);
        Uri pluginUri = Uri.parse(uri.getQueryParameter(KEY_URI));
        if (provider != null) {
            return provider.query(pluginUri, projection, selection, selectionArgs, sortOrder);
        }

        return null;
    }
```

可以看出 RemoteContentProvider 的 query，首先调用了 getContentProvider 找到真正的插件 Provider。

```java
    private ContentProvider getContentProvider(final Uri uri) {
        ...
        synchronized (sCachedProviders) {
            LoadedPlugin plugin = pluginManager.getLoadedPlugin(uri.getQueryParameter(KEY_PKG));
            if (plugin == null) {
                try {
                    pluginManager.loadPlugin(new File(uri.getQueryParameter(KEY_PLUGIN)));
                } catch (Exception e) {
                    Log.w(TAG, e);
                }
            }

            final ProviderInfo providerInfo = pluginManager.resolveContentProvider(auth, 0);
            if (providerInfo != null) {
                RunUtil.runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            LoadedPlugin loadedPlugin = pluginManager.getLoadedPlugin(uri.getQueryParameter(KEY_PKG));
                            ContentProvider contentProvider = (ContentProvider) Class.forName(providerInfo.name).newInstance();
                            contentProvider.attachInfo(loadedPlugin.getPluginContext(), providerInfo);
                            sCachedProviders.put(auth, contentProvider);
                        } catch (Exception e) {
                            Log.w(TAG, e);
                        }
                    }
                }, true);
                return sCachedProviders.get(auth);
            }
        }

        return null;
    }
```

可以看出 getContentProvider 方法把之前组合好的 uri 通过 getQueryParameter 的方式查到里面携带的信息参数。

```java
pluginManager.loadPlugin(new File(uri.getQueryParameter(KEY_PLUGIN)));
```

比如用来从指定地址加载插件。

然后用 Class.forName 构造了 ContentProvider，attach 了 providerInfo，并且做了 provider 缓存。

```java
ContentProvider contentProvider = (ContentProvider) Class.forName(providerInfo.name).newInstance();
contentProvider.attachInfo(loadedPlugin.getPluginContext(), providerInfo);
```

至此分发到了插件的 ContentProvider 并调用了插件 Provider 的方法。

### BroadcastReceiver 的支持

BroadcastReceiver 的支持比较简单，直接将插件 AndroidManifest 中的静态广播转为动态广播，插件原有的动态广播放入宿主动态注册。

```java
    public LoadedPlugin(PluginManager pluginManager, Context context, File apk) throws Exception {
        ...
        // Register broadcast receivers dynamically
        Map<ComponentName, ActivityInfo> receivers = new HashMap<ComponentName, ActivityInfo>();
        for (PackageParser.Activity receiver : this.mPackage.receivers) {
            receivers.put(receiver.getComponentName(), receiver.info);
    
            BroadcastReceiver br = BroadcastReceiver.class.cast(getClassLoader().loadClass(receiver.getComponentName().getClassName()).newInstance());
            for (PackageParser.ActivityIntentInfo aii : receiver.intents) {
                this.mHostContext.registerReceiver(br, aii);
            }
        }
        ...
```

这一步是在 LoadedPlugin 的构造方法中解析 BroadcastReceiver 完成的。

可以看出，对于插件 APK 静态注册的某些广播，比如开机广播，宿主 APK 是无法实现其功能的，因为它们被转换为了动态广播，手机开机时，应用进程没有被拉起，无法动态监听开机广播。

## 总结

1. VirtualAPK 加载插件后会得到一个 LoadedPlugin，它是插件的核心类，类似于 Android 自身的 LoadedApk。LoadedPlugin 通过 insertDex 方法把插件 APK 的 Dex 和宿主 APK 的 Dex 合并，这样宿主 APK 包含了所有的 Dex，宿主可以加载插件 APK 的类。 同时 LoadedPlugin 的 DexClassLoader 在构造时传入了宿主的 classLoader，所以插件可以加载宿主的类。

2. VirtualAPK 将宿主的 resource 和插件的 resource 合并，最后把得到的新 Resources 更新给了每个插件的 LoadedPlugin。所以宿主 APK 和 插件 APK 可以相互共享资源。

3. VirtualAPK 通过“欺上瞒下”的方式分别 hook 了 Instrumentation、ActivityManager、IContentProvider，并将原始的调用 hook 为对占坑的 StubActivity、LocalService、RemoteContentProvider 的调用，在重新分发的过程中又解析了 intent 和 uri，并且加载了插件中的对应组件，调用原始组件的方法。BroadcastReceiver 和其他三个组件的方式有所不同，采用了静态广播转动态广播的方法。