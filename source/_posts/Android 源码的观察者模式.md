# Android 源码的观察者模式

## 观察者模式介绍

观察者模式是一个使用率非常高的模式，它最常用的地方是 GUI 系统、订阅-发布系统。因为这个模式的一个非常重要的作用就是解耦，将被观察者和观察者解耦，使得它们之间的依赖性更小，甚至做到毫无依赖。以 GUI 系统来说，应用的 UI 具有易变性，尤其是前期随着业务的改变或者产品的需求修改，应用界面也会经常性变化，但是业务逻辑基本变化不大，此时，GUI 系统需要一套机制来应对这种情况，使得 UI 层与具体的业务逻辑解耦，观察者模式此时就派上用场了。

## 观察者模式的定义

定义对象间一种一对多的依赖关系，使得每当一个对象改变状态，则所有依赖于它的对象都会得到通知并被自动更新。

## ListView 的 notifyDataSetChanged

ListView 是 Android 中最重要的控件之一，而 ListView 最重要的一个功能就是 Adapter。通常，在我们往 ListView 添加数据后，都会调用 Adapter 的 notifyDataSetChanged 方法。

notifyDataSetChanged 方法定义在 BaseAdapter 中，具体代码如下：

```java
public abstract class BaseAdapter implements ListAdapter, SpinnerAdapter {
    // 数据集观察者
    private final DataSetObservable mDataSetObservable = new DataSetObservable();
    private CharSequence[] mAutofillOptions;

    public boolean hasStableIds() {
        return false;
    }
    // 注册观察者
    public void registerDataSetObserver(DataSetObserver observer) {
        mDataSetObservable.registerObserver(observer);
    }
    // 反注册观察者
    public void unregisterDataSetObserver(DataSetObserver observer) {
        mDataSetObservable.unregisterObserver(observer);
    }
    
    // 当数据集变化时，通知所有观察者
    /**
     * Notifies the attached observers that the underlying data has been changed
     * and any View reflecting the data set should refresh itself.
     */
    public void notifyDataSetChanged() {
        mDataSetObservable.notifyChanged();
    }
...
}
```

一看 BaseAdapter 代码就大体有了这么一个认识：BaseAdapter 是一个观察者模式！

mDataSetObservable.notifyChanged 方法如下：

```java
// 数据集观察者
public class DataSetObservable extends Observable<DataSetObserver> {
    // 调用每个观察者的 onChange 函数来通知它们被观察者发生了变化
    /**
     * Invokes {@link DataSetObserver#onChanged} on each observer.
     * Called when the contents of the data set have changed.  The recipient
     * will obtain the new contents the next time it queries the data set.
     */
    public void notifyChanged() {
        synchronized(mObservers) {
            // 调用所有观察者的 onChanged 方法
            // since onChanged() is implemented by the app, it could do anything, including
            // removing itself from {@link mObservers} - and that could cause problems if
            // an iterator is used on the ArrayList {@link mObservers}.
            // to avoid such problems, just march thru the list in the reverse order.
            for (int i = mObservers.size() - 1; i >= 0; i--) {
                mObservers.get(i).onChanged();
            }
        }
    }

    /**
     * Invokes {@link DataSetObserver#onInvalidated} on each observer.
     * Called when the data set is no longer valid and cannot be queried again,
     * such as when the data set has been closed.
     */
    public void notifyInvalidated() {
        synchronized (mObservers) {
            for (int i = mObservers.size() - 1; i >= 0; i--) {
                mObservers.get(i).onInvalidated();
            }
        }
    }
}
```

这个代码很简单，就是在 mDataSetObservable 的 notifyChanged 中遍历所有观察者，并且调用它们的 onChanged 方法，从而告知观察者发生了变化。

那么这些观察者是从哪里来的呢？其实这些观察者就是 ListView 通过 setAdapter 方法设置 Adapter 产生的。

ListView 的 setAdapter 方法如下：

```java
@Override
    public void setAdapter(ListAdapter adapter) {
        // 如果已经有了一个 Adapter，那么先注销该 Adapter 对应的观察者
        if (mAdapter != null && mDataSetObserver != null) {
            mAdapter.unregisterDataSetObserver(mDataSetObserver);
        }

        ...
        super.setAdapter(adapter);

        if (mAdapter != null) {
            mAreAllItemsSelectable = mAdapter.areAllItemsEnabled();
            mOldItemCount = mItemCount;
            // 获取数据的数量
            mItemCount = mAdapter.getCount();
            checkFocus();
            // 创建一个数据观察者
            mDataSetObserver = new AdapterDataSetObserver();
            // 将这个观察者注册到 Adapter 中，实际上是注册到 DataSetObservable 中
            mAdapter.registerDataSetObserver(mDataSetObserver);

            mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());

           ...

        requestLayout();
    }
```

从程序中可以看到，在设置 Adapter 时会构建一个 AdapterDataSetObserver，这就是上面所说的观察者，最后，将这个观察者注册到 Adapter 中，这样我们的被观察者、观察者都有了。

AdapterDataSetObserver 定义在 ListView 的父类 AbsListView 中，具体代码如下：

```java
    class AdapterDataSetObserver extends AdapterView<ListAdapter>.AdapterDataSetObserver {
        @Override
        public void onChanged() {
            super.onChanged();
            if (mFastScroll != null) {
                mFastScroll.onSectionsChanged();
            }
        }

        @Override
        public void onInvalidated() {
            super.onInvalidated();
            if (mFastScroll != null) {
                mFastScroll.onSectionsChanged();
            }
        }
    }
```

AdapterDataSetObserver 继承自 AbsListView 的父类 AdapterView 的 AdapterDataSetObserver。

```java
    class AdapterDataSetObserver extends DataSetObserver {

        private Parcelable mInstanceState = null;

        @Override
        public void onChanged() {
            mDataChanged = true;
            // 获取 Adapter 中数据的数量
            mOldItemCount = mItemCount;
            mItemCount = getAdapter().getCount();

            // Detect the case where a cursor that was previously invalidated has
            // been repopulated with new data.
            if (AdapterView.this.getAdapter().hasStableIds() && mInstanceState != null
                    && mOldItemCount == 0 && mItemCount > 0) {
                AdapterView.this.onRestoreInstanceState(mInstanceState);
                mInstanceState = null;
            } else {
                rememberSyncState();
            }
            checkFocus();
            // 重新布局 ListView、GridView 等 AdapterView 组件
            requestLayout();
        }

        @Override
        public void onInvalidated() {
            mDataChanged = true;

            if (AdapterView.this.getAdapter().hasStableIds()) {
                // Remember the current state for the case where our hosting activity is being
                // stopped and later restarted
                mInstanceState = AdapterView.this.onSaveInstanceState();
            }

            // Data is invalid so we should reset our state
            mOldItemCount = mItemCount;
            mItemCount = 0;
            mSelectedPosition = INVALID_POSITION;
            mSelectedRowId = INVALID_ROW_ID;
            mNextSelectedPosition = INVALID_POSITION;
            mNextSelectedRowId = INVALID_ROW_ID;
            mNeedSync = false;

            checkFocus();
            requestLayout();
        }

        public void clearSavedState() {
            mInstanceState = null;
        }
    }

```

当 ListView 的数据发生变化时，调用 Adapter 的 notifyDataSetChanged 函数，这个函数会调用 DataSetObservable 的 notifyChanged 函数。notifyChanged 会调用所有观察者（AdapterDataSetObserver）的 onChanged 方法，在 onChanged 方法中又会调用 ListView 重新布局的函数使得 ListView 刷新界面。

## BroadcastReceiver 广播接收器

BroadcastReceiver 是 Android 的四大组件之一，是一个典型的发布-订阅模式，也就是观察者模式。


### 注册广播

当调用 registerReceiver 注册广播时，会调用 ContextImpl 的 registerReceiverInternal 方法。

registerReceiverInternal 代码如下：

```java
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context, int flags) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    // 获取 Handler 来投递消息
                    scheduler = mMainThread.getHandler();
                }
                // 获取 IIntentReceiver 对象，通过它与 AMS 交互，并且通过 Handler 传递消息
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            final Intent intent = ActivityManager.getService().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                    broadcastPermission, userId, flags);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

```

mMainThread.getHandler() 会获取主线程的 Handler，这个 Handler 是用来分发 AMS 发送过来的广播。也就是 ActivityThread 里面名叫 H 的广播。

ActivityThread 的 getHandler 方法如下：

```java
    final Handler getHandler() {
        return mH;
    }
```

ContextImpl 的 registerReceiverInternal 方法通过 mPackageInfo 的 getReceiverDispatcher 获取一个 IIntentReceiver 对象 rd，它是一个 Binder 对象，接下来会把它传给 AMS。AMS 在收到相应的广播时，就是通过这个 Binder 对象来通知 Activity 里面的接收者。

AMS 的 registerReceiver 方法如下：

```java
    public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId,
            int flags) {
        ...
        // 获取 ProcessRecord
        ProcessRecord callerApp = null;
        ...
        // 根据 Intent 查找匹配的 sticky 接收器（粘性广播）
        ArrayList<Intent> allSticky = null;
        if (stickyIntents != null) {
            ...
        }
        ...
        synchronized (this) {
            ...
            // 获取 ReceiverList
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            ...
            // 构建 BroadcastFilter，并且添加到 ReceiverList 中
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId, instantApp, visibleToInstantApps);
            ...
                mReceiverResolver.addFilter(bf);
            ...
            return sticky;
        }
    }
```

这里首先查找有没有对应的 sticky intent 列表才能在。sticky intent 是粘性广播，我们在最后一次调用 sendStickyBroadcast 来发送某个广播时，系统会把代表这个广播的 intent 保存下来，这样，后来调用 registerReceiver 来注册相同 action 的广播接收器时，就会得到这个最后发出的广播。这个最后发出的广播虽然被处理完了，但是仍然被粘在 AMS，以便下个注册相应 Action 的广播接收器还能继续处理。

接下来会把 receiver 保存在 ReceiverList 中，然后创建 BroadcastFilter 把广播接收器列表 rl 和 filter 关联起来，保存在 mReceiverResolver。

### 发送广播

sendBroadcast 发送广播时，会先通过 Binder 调用到 AMS 的 broadcastIntentLocked 方法。这里会通过 action 找到对应的广播接收器，然后把广播放到消息队列中，完成第一个阶段对广播的异步分发。

```java
    @GuardedBy("this")
    final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {
        intent = new Intent(intent);
        ...
                queue.enqueueOrderedBroadcastLocked(r);
                // 广播的异步分发
                queue.scheduleBroadcastsLocked();
        ...

        return ActivityManager.BROADCAST_SUCCESS;
    }

```

AMS 在消息循环中处理这个广播，并通过 Binder 把这个广播分发给 ReceiverDispatcher，ReceiverDispatcher 把这个广播放到 ActivityThread 的消息队列中。

scheduleBroadcastsLocked 方法：

```java
    public void scheduleBroadcastsLocked() {
        ...
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;
    }
```

BroadcastHandler 处理广播消息：

```java
    private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BROADCAST_INTENT_MSG: {
                    if (DEBUG_BROADCAST) Slog.v(
                            TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
                    processNextBroadcast(true);
                } break;
                case BROADCAST_TIMEOUT_MSG: {
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
            }
        }
    }
```
BroadcastHandler 会调用 processNextBroadcastLocked 处理广播。

processNextBroadcastLocked 方法如下：

```java
    final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
        BroadcastRecord r;
        ...
                        performReceiveLocked(r.callerApp, r.resultTo,
                            new Intent(r.intent), r.resultCode,
                            r.resultData, r.resultExtras, false, false, r.userId);
        ...
    }
```

performReceiveLocked 方法会调用到 ActivityThread 的 scheduleRegisteredReceiver 方法。所以，注册广播的应用就收到了广播。

```java
    void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
            Intent intent, int resultCode, String data, Bundle extras,
            boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        ...
                    app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                            data, extras, ordered, sticky, sendingUser, app.repProcState);
        ...
    }
```

scheduleRegisteredReceiver 在 ActivityThread 类。

```java
        public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                int resultCode, String dataStr, Bundle extras, boolean ordered,
                boolean sticky, int sendingUser, int processState) throws RemoteException {
            updateProcessState(processState, false);
            receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                    sticky, sendingUser);
        }
```

IIntentReceiver 定义在 LoadedApk，IIntentReceiver 的 performReceive 如下：

```java
        public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            final Args args = new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
            ...
            if (intent == null || !mActivityThread.post(args.getRunnable())) {
                if (mRegistered && ordered) {
                    IActivityManager mgr = ActivityManager.getService();
                    ...
                    args.sendFinished(mgr);
                }
            }
        }

    }
```

performReceive 构造了一个 Args 对象，然后 post 给 Handler，然后告诉 AMS 发送完毕。

Args 的 Runnable 如下：

```java
        final class Args extends BroadcastReceiver.PendingResult {
            ...
            public final Runnable getRunnable() {
                return () -> {
                    ...
                        receiver.onReceive(mContext, intent);
                    ...
                };
            }
        }

```

可以看出 runnable 调用了  receiver.onReceive，也就是 BroadcastReceiver 的 onReceive 回调。因为 runnable 是 post 到 mActivityThread，所以 onReceive 也是执行在主线程。

简单来说，广播是一个发布-订阅过程，也是观察者模式。订阅过程将广播注册到 AMS 的广播列表中。发布过程通过查询广播的 intentfilter 对应的 broadcastReceiver，然后通过 ReceiverDisptacher 将广播分发给各个订阅对象，从而完成这个发布-订阅过程。
