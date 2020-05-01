# Android 源码的外观模式

## 外观模式介绍

外观模式（Facade）在开发过程中的运用频率非常高，尤其是在现阶段各种第三方 SDK 充斥在我们的周边，而这些 SDK 很大概率会使用外观模式。通过一个外观类使得整个系统的接口只有一个统一的高层接口，这样能够降低用户的使用成本，也对用户屏蔽了很多实现细节。当然，在我们的开发过程中，外观模式也是我们封装 API 的常用手段，例如网络模块、ImageLoader模块等。

## 外观模式定义

要求一个子系统的外部与其内部的通信必须通过一个统一的对象进行。外观模式（Facade 模式）提供一个高层次的接口，使得子系统更易于使用。

## Android 源码中的外观模式

在用 Android 开发过程中，Context 是最重要的一个类型。Context 只是一个抽象类，它的真正实现在 ContextImpl 类中，ContextImpl 就是今天我们要分析的外观类。

在应用启动时，首先会 fork 一个子进程，并且调用 ActivityThread 的 main 方法启动该进程。ActivityThread 又会构建 Application 对象，然后和 Activity、ContextImpl 关联起来，最后会调用 Activity 的 onCreate、onStart、onResume 函数使 Activity 运行起来，此时应用的用户界面就呈现在我们面前。

main 函数会间接地调用 ActivityThread 中的 handleLaunchActivity 函数启动默认的 Activity。

handleLaunchActivity 方法如下：

```java
    @Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        ...
        // 创建并且加载 Activity，调用它的 onCreate
        final Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            if (!r.activity.mFinished && pendingActions != null) {
                pendingActions.setOldState(r.state);
                pendingActions.setRestoreInstanceState(true);
                pendingActions.setCallOnPostCreate(true);
            }
        } else {
            // If there was an error, for any reason, tell the activity manager to stop us.
            try {
                ActivityManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
    }
```

performLaunchActivity 方法如下：

```java
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        // 构建 ContextImpl
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            // 创建 Activity
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            // 创建 Application
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

            if (activity != null) {
                // 获取 Activity 的 title
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                // Activity 与 Context、Application 关联起来
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    // 回调 Activity 的 onCreate 方法
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
            }
            r.setState(ON_CREATE);

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }

```

在 handleLaunchActivity 中会调用 perfromLaunchActivity 执行 Applicaton、ContextImpl、Activity 的创建工作，并且通过Activity 的 attach 将这3者关联起来，

Activity 是 Context 的子类，因此，Activity 就具有了 Context 定义的所有方法。但 Activity 并不实现具体的功能，它只是继承了 Context 的接口，并且将相关的操作交给 ContextImpl。ContextImpl 存储在 Activity 的上两层父类 ContextWrapper 中，变量名为 mBase，具体代码如下：

ContextThemeWrapper

```java
public class ContextThemeWrapper extends ContextWrapper {
    ...
    @Override
    protected void attachBaseContext(Context newBase) {
        super.attachBaseContext(newBase);
    }
    ...
```

ContextWrapper

```java
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    
    /**
     * Set the base context for this ContextWrapper.  All calls will then be
     * delegated to the base context.  Throws
     * IllegalStateException if a base context has already been set.
     * 
     * @param base The new base context for this wrapper.
     */
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    ...
```

在 ActivityThread 类的 perfromLaunchActivity 函数中会调用 Activity 的 attach 方法将 ContextImpl 等对象关联到 Activity 中，这个 ContextImp l最终会被 ContentWrapper 类的 mBase 字段引用。

Activity 的 attach 如下：

```java
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        // 调用自身的 attachBaseContext
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);

        setAutofillCompatibilityEnabled(application.isAutofillCompatibilityEnabled());
        enableAutofillCompatibilityIfNeeded();
    }
```

attach 主要就是一些赋值操作。在 attach 中，调用了 attachBaseContext 函数。attachBaseContext 调用了父类 ContextWrapper 类，它就是简单地将 Context 参数传递给 mBase 字段。此时，我们的 Activity 内部就持有了 ContextImpl 的引用。

Activity 的 attachBaseContext

```java
    @Override
    protected void attachBaseContext(Context newBase) {
        // 调用了 ContextThemeWrapper 的 attachBaseContext
        super.attachBaseContext(newBase);
        if (newBase != null) {
            newBase.setAutofillClient(this);
        }
    }
```

Activity 在开发过程中部分充当了代理的角色，例如，当我们通过 Activity 对象调用 sendBroadcast、getResource 等函数时，实际上 Activity 只是代理了 ContextImpl 的操作，也就是内部都调用了 mBase 对象的相应方法来处理，这些方法被封装在 Activity 的父类 ContextWrapper 中。

ContextWrapper

```java
public class ContextWrapper extends Context {
    Context mBase;
    ...
    @Override
    public void sendBroadcast(Intent intent) {
        mBase.sendBroadcast(intent);
    }
    ...
    @Override
    public Resources getResources() {
        return mBase.getResources();
    }

    @Override
    public PackageManager getPackageManager() {
        return mBase.getPackageManager();
    }
    ...
}
```

它的实现如下：

ContextImpl

```java
class ContextImpl extends Context {
    ...
    @Override
    public void sendBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    ...
    @Override
    public Resources getResources() {
        return mResources;
    }

    @Override
    public PackageManager getPackageManager() {
        if (mPackageManager != null) {
            return mPackageManager;
        }

        IPackageManager pm = ActivityThread.getPackageManager();
        if (pm != null) {
            // Doesn't matter if we make more than one instance.
            return (mPackageManager = new ApplicationPackageManager(this, pm));
        }

        return null;
    }
    ...
}
```

ContextImpl 内部封装了很多不同子系统的操作，例如，Activity 的跳转、发送广播、启动服务、设置壁纸等，这些工作并不是在 ContextImpl 中实现，而是转交给了具体的子系统进行处理。通过 Context 这个抽象类定义了一组接口，ContextImpl 实现 Context 定义的接口，使得用户可以通过 Context 这个接口统一与 Android 系统进行交互，这样用户通常情况下就不需要对每个子系统进行了解，例如启动 Activity 时用户不需要手动调用 mMainThread.getInstrumentation().execStartActivity 启动 Activity。用户与系统服务的交互都通过 Context 的高层接口。这样对用户屏蔽了具体实现的细节，降低了使用成本。

## 参考资料

《Android 源码设计模式解析与实战 · 第23章 统一编程接口——外观模式》