# 自定义 ActivityNotifier 观察所有 Activity 切换

我们知道，使用 Application 注册 ActivityLifecycleCallbacks 可以观察到本应用的所有 Activity 的生命周期切换。那么有没有一种方法可以观察到手机上所有应用的 Activity 的生命周期切换？

因为所有 Activity 的生命周期回调都会经过 ActivityManagerService(AMS) 管理，最后回调到应用的 ActivityThread。所以如果想要观察手机上所有应用的 Activity 生命周期回调，最好的办法就是在 AMS 中添加自定义的接口来实现。

## ActivityTaskManagerService

Android 10 版本（Q 版本）中新增了一个 ActivityTaskManagerService（ATMS） 类，它的作用是承担 AMS 的部分功能。由于 AMS 过于庞大，所以将部分功能剥离给 ATMS 来处理。

ActivityTaskManagerService 中有一些回调方法，例如 activityResumed 方法，所有 Activity 的 onResume 走完之后，都会调用 ATMS 的 activityResumed。

```java
    @Override
    public final void activityResumed(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized (mGlobalLock) {
            ActivityRecord.activityResumedLocked(token);
            mWindowManager.notifyAppResumedFinished(token);
        }
        Binder.restoreCallingIdentity(origId);
    }

    @Override
    public final void activityTopResumedStateLost() {
        ...
    }

    @Override
    public final void activityPaused(IBinder token) {
        ...
    }

    @Override
    public final void activityStopped(IBinder token, Bundle icicle,
            PersistableBundle persistentState, CharSequence description) {
        ...
    }

    @Override
    public final void activityDestroyed(IBinder token) {
        ...
    }

    @Override
    public final void activityRelaunched(IBinder token) {
        ...
    }
```

知道了 ActivityTaskManagerService 的 activityResumed 这些系统回调后，我们就可以在每个方法中添加分发方法，将每个生命周期事件分发出去。如果发现有应用注册了某个事件，就将该事件分发给它。

比如自定义接口 IActivityNotifier

```
interface IActivityNotifier {
    void call(Bundle extras);
}
```

call 方法表示某个事件被触发，extras 参数表示这个事件携带的一些信息，比如是什么原因引起的这次事件。

然后通过 AMS 注册接口。

```java
    public void regsiterActivityNotifier(IActivityNotifier notifier) {
        ActivityManagerEx.regsiterActivityNotifier(notifier);
    }
```

这里的 ActivityManagerEx 表示 ActivityManager 的扩展类，它包含了一些定制的方法。

然后使用 RemoteCallbackList 保存每个注册过来的 IActivityNotifier 回调。

```java
    private final RemoteCallbackList<IActivityNotifier> mCallbacks =
            new RemoteCallbackList<>();
```

分别注册和反注册回调。

```java
    public void regsiterActivityNotifier(IActivityNotifier cb) {
        if (cb != null) {
            mCallbacks.register(cb);
        }
    }

    public void unregsiterActivityNotifier(IActivityNotifier cb) {
        if (cb != null) {
            mCallbacks.unregister(cb);
        }
    }
```

最后在 ATMS 的 activityResumed 将事件分发出去。

```java
    @Override
    public final void activityResumed(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized (mGlobalLock) {
            ActivityRecord.activityResumedLocked(token);
            mWindowManager.notifyAppResumedFinished(token);
            // 分发事件
            ATMSEx.dispatchActivityLifeState(...);
        }
        Binder.restoreCallingIdentity(origId);
    }
```

dispatchActivityLifeState 方法将事件通过消息的方式分发出去：

```java
    public void dispatchActivityLifeState() {
        Message message = Message.obtain(mHandler, NOTIFY_CALL);
        Bundle bundle = new Bundle();
        // 存入数据
        bundle.putInt(...);
        message.obj = bundle;
        mHandler.sendMessage(message);
    }
```

最后在 mHandler 中处理消息，遍历每一个 IActivityNotifier 回调，依次派发。

比如：

```java
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case NOTIFY_CALL: {
                    int n = mCallbacks.beginBroadcast();
                    for (int i = 0; i < n; ++i) {
                        try {
                            mCallbacks.getBroadcastItem(i).call(bundle);
                        } catch (RemoteException e) {
                        }
                    }
                    mCallbacks.finishBroadcast();
                    break;
                }
            }
        }
    };
```

可以看出，整个流程和 Application 注册 ActivityLifecycleCallbacks 的过程很类似，都是在某个事件发生时派发给各个注册的接口。只不过 IActivityNotifier 是应用注册到 AMS。而 ActivityLifecycleCallbacks 在应用内部注册。

## ActivityLifecycleItem

最后看一下为什么所有 Activity 的 onResume 后，都会走到 ATMS 的 activityResumed 方法。

在 Android 9 版本（P 版本）之后，AMS 不是直接调用 ActivityThread 的各个方法，而是通过 ActivityLifecycleItem 来管理。比如 Acivity 启动后最终要到 onResume 状态，就使用 ResumeActivityItem。

```java
/**
 * Request to move an activity to resumed state.
 * @hide
 */
public class ResumeActivityItem extends ActivityLifecycleItem {

    private static final String TAG = "ResumeActivityItem";

    private int mProcState;
    private boolean mUpdateProcState;
    private boolean mIsForward;

    @Override
    public void preExecute(ClientTransactionHandler client, IBinder token) {
        if (mUpdateProcState) {
            client.updateProcessState(mProcState, false);
        }
    }

    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
        client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                "RESUME_ACTIVITY");
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        try {
            // TODO(lifecycler): Use interface callback instead of AMS.
            ActivityTaskManager.getService().activityResumed(token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
```

和 AsyncTask 类似，ResumeActivityItem 有 3 个回调方法，preExecute、execute、postExecute，依次表示执行前、执行中、执行后。

查看 execute 和 postExecute 方法，可以看出 handleResumeActivity 之后会执行 ATMS 的 activityResumed 方法。