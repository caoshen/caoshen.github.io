# Android setPackagesSuspended 暂停应用

setPackagesSuspended 是 PackageManager 的一个 public 方法，它可以用来暂停应用。
应用被暂停之后会进入 Suspended 状态，无法点击打开，会弹出一个系统对话框，提示应用已被暂停。应用的后台活动比如播放音乐也会被暂停。
Android 数字健康（com.google.android.apps.wellbeing)就是利用 setPackagesSuspended 方法暂停应用，达到限制应用使用的效果。

## setPackagesSuspended

setPackagesSuspended 方法如下：

```java
    @SystemApi
    @RequiresPermission(Manifest.permission.SUSPEND_APPS)
    public String[] setPackagesSuspended(String[] packageNames, boolean suspended,
            @Nullable PersistableBundle appExtras, @Nullable PersistableBundle launcherExtras,
            String dialogMessage) {
        throw new UnsupportedOperationException("setPackagesSuspended not implemented");
    }
```

可以看出 setPackagesSuspended 是系统 api 方法，而且需要在 manifest 定义权限 Manifest.permission.SUSPEND_APPS。
Manifest.permission.SUSPEND_APPS 权限也是系统级别的。普通应用没有权限，只用系统应用可以使用 setPackagesSuspended。同时 setPackagesSuspended 也是 hide 方法。

setPackagesSuspended 的使用方法如下：

```java
getPackageManager().setPackagesSuspended(packageNames, true, null, null, dialogMessage);
```

getPackageManager 是 Context 类的抽象方法，本质上是调用了 ContextImpl 的 getPackageManager 方法。

getPackageManager() 方法如下：

```java
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
```

可以看出构造了一个 ApplicationPackageManager，传入 context 和 IPackageManager。IPackageManager 是一个 binder 接口，从 ServiceManager 通过 getService 获取。

## ApplicationPackageManager setPackagesSuspended

查看 ApplicationPackageManager 的 setPackagesSuspended 方法。代码如下：

```java
    @Override
    public String[] setPackagesSuspended(String[] packageNames, boolean suspended,
            PersistableBundle appExtras, PersistableBundle launcherExtras,
            String dialogMessage) {
        try {
            return mPM.setPackagesSuspendedAsUser(packageNames, suspended, appExtras,
                    launcherExtras, dialogMessage, mContext.getOpPackageName(),
                    mContext.getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

可以看出远程调用了 PMS 的 setPackagesSuspendedAsUser，传入了应用自身的包名和 uid。

## PMS 的 setPackagesSuspendedAsUser

PMS 的 setPackagesSuspendedAsUser 方法如下：

```java
    @Override
    public String[] setPackagesSuspendedAsUser(String[] packageNames, boolean suspended,
            PersistableBundle appExtras, PersistableBundle launcherExtras,
            SuspendDialogInfo dialogInfo, String callingPackage, int userId) {
            for (int i = 0; i < packageNames.length; i++) {
            ...
            synchronized (mPackages) {
                pkgSetting.setSuspended(suspended, callingPackage, dialogInfo, appExtras,
                        launcherExtras, userId);
            }
        ...
        if (!changedPackagesList.isEmpty()) {
            final String[] changedPackages = changedPackagesList.toArray(
                    new String[changedPackagesList.size()]);V
            sendPackagesSuspendedForUser(
                    changedPackages, changedUids.toArray(), userId, suspended, launcherExtras);
            sendMyPackageSuspendedOrUnsuspended(changedPackages, suspended, appExtras, userId);
            synchronized (mPackages) {
                scheduleWritePackageRestrictionsLocked(userId);
            }
        }
        return unactionedPackages.toArray(new String[unactionedPackages.size()]);
    }
```

可以看出以上代码调用了 pkgSetting.setSuspended，然后调用了 sendPackagesSuspendedForUser、sendMyPackageSuspendedOrUnsuspended。
setPackagesSuspendedAsUser 返回了 unactionedPackages，也就是没有被 suspend 的应用。

pkgSetting 的 setSuspended 方法其实就是 PackageSettingBase 的 setSuspended。

PackageSettingBase 的 setSuspended 方法会将 PackageUserState 的 suspended 改变。

```java
existingUserState.suspended = suspended;
```

在 PackageParser 的 updateApplicationInfo 会更新应用的 applicationInfo。

```java
        if (state.suspended) {
            ai.flags |= ApplicationInfo.FLAG_SUSPENDED;
        } else {
            ai.flags &= ~ApplicationInfo.FLAG_SUSPENDED;
        }
```

可以看出会根据 PackageUserState 的 suspended 属性添加或者删除 ApplicationInfo.FLAG_SUSPENDED。

FWK 中的其他地方判断一个应用是否被暂停了就是依靠这个 ApplicationInfo.FLAG_SUSPENDED 标记。

比如 ActivityStartInterceptor 应用启动拦截的 interceptSuspendedPackageIfNeeded 方法。

```java
        // Do not intercept if the package is not suspended
        if (mAInfo == null || mAInfo.applicationInfo == null ||
                (mAInfo.applicationInfo.flags & FLAG_SUSPENDED) == 0) {
            return false;
        }
```

如果应用没有 ApplicationInfo.FLAG_SUSPENDED（mAInfo.applicationInfo.flags & FLAG_SUSPENDED），说明它没有被暂停，不要拦截它的启动。

## sendPackagesSuspendedForUser

sendPackagesSuspendedForUser 代码如下：

```java
private void sendPackagesSuspendedForUser(String[] pkgList, int[] uidList, int userId,
            boolean suspended, PersistableBundle launcherExtras) {
        final Bundle extras = new Bundle(3);
        extras.putStringArray(Intent.EXTRA_CHANGED_PACKAGE_LIST, pkgList);
        extras.putIntArray(Intent.EXTRA_CHANGED_UID_LIST, uidList);
        if (launcherExtras != null) {
            extras.putBundle(Intent.EXTRA_LAUNCHER_EXTRAS,
                    new Bundle(launcherExtras.deepCopy()));
        }
        sendPackageBroadcast(
                suspended ? Intent.ACTION_PACKAGES_SUSPENDED
                        : Intent.ACTION_PACKAGES_UNSUSPENDED,
                null, extras, Intent.FLAG_RECEIVER_REGISTERED_ONLY, null, null,
                new int[] {userId}, null);
}
```

可以看出以上代码将参数封装到 bundle 中，然后发送了广播。广播 Action 是 Intent.ACTION_PACKAGES_SUSPENDED 或者 Intent.ACTION_PACKAGES_UNSUSPENDED。

sendPackageBroadcast 会调用 doSendBroadcast，然后调用 AMS 的 broadcastIntent。

## sendMyPackageSuspendedOrUnsuspended

sendMyPackageSuspendedOrUnsuspended 方法如下：

```java
    private void sendMyPackageSuspendedOrUnsuspended(String[] affectedPackages, boolean suspended,
            PersistableBundle appExtras, int userId) {
        ...
        if (suspended) {
            action = Intent.ACTION_MY_PACKAGE_SUSPENDED;
            if (appExtras != null) {
                final Bundle bundledAppExtras = new Bundle(appExtras.deepCopy());
                intentExtras.putBundle(Intent.EXTRA_SUSPENDED_PACKAGE_EXTRAS, bundledAppExtras);
            }
        } else {
            action = Intent.ACTION_MY_PACKAGE_UNSUSPENDED;
        }
        ...
    }
```

sendMyPackageSuspendedOrUnsuspended 也是发送广播，只不过它发送了 Intent.ACTION_MY_PACKAGE_SUSPENDED 和 Intent.ACTION_MY_PACKAGE_UNSUSPENDED。
被暂停的应用可以通过注册这两个 Action 的广播来通知自己是否被暂停了。另外被暂停的应用也可以使用 isPackageSuspended 方法判断自己是否处于暂停状态。

## ACTION_PACKAGES_SUSPENDED 广播接收

查看 AMS 的 broadcastIntentLocked 方法，里面对 ACTION_PACKAGES_SUSPENDED 广播做了处理：

```java
                        case Intent.ACTION_PACKAGES_SUSPENDED:
                        case Intent.ACTION_PACKAGES_UNSUSPENDED:
                            final boolean suspended = Intent.ACTION_PACKAGES_SUSPENDED.equals(
                                    intent.getAction());
                            final String[] packageNames = intent.getStringArrayExtra(
                                    Intent.EXTRA_CHANGED_PACKAGE_LIST);
                            final int userHandle = intent.getIntExtra(
                                    Intent.EXTRA_USER_HANDLE, UserHandle.USER_NULL);

                            mAtmInternal.onPackagesSuspendedChanged(packageNames, suspended,
                                    userHandle);
                            break;
```

可以看出以上代码调用了 mAtmInternal.onPackagesSuspendedChanged 方法。

mAtmInternal 是ActivityTaskManagerService (ATMS)，它调用了 RecentTasks 的 onPackagesSuspendedChanged 方法。

```java
        @Override
        public void onPackagesSuspendedChanged(String[] packages, boolean suspended, int userId) {
            synchronized (mGlobalLock) {
                mRecentTasks.onPackagesSuspendedChanged(packages, suspended, userId);
            }
        }
```

RecentTasks 的 onPackagesSuspendedChanged 方法如下：

```java
    void onPackagesSuspendedChanged(String[] packages, boolean suspended, int userId) {
         ...
                if (suspended) {
                    mSupervisor.removeTaskByIdLocked(tr.taskId, false,
                            REMOVE_FROM_RECENTS, "suspended-package");
                }
         ...
    }
```

可以看出如果应用被暂停，它会从 tasks 中移除。

除了 AMS， NMS（NotificationManagerService）、WMS（WindowManagerService）和其他一些地方也注册了 ACTION_PACKAGES_SUSPENDED 广播。

## 总结

1. PacakgeManager 的 setPackagesSuspended 方法可以暂停或者恢复应用。
2. setPackagesSuspended 通过 PMS 改变 ApplicationInfo 的 FLAG_SUSPENDED 标记和发送 ACTION_PACKAGES_SUSPENDED、ACTION_MY_PACKAGE_SUSPENDED 通知 FWK 的其他服务（AMS 等）实现应用暂停的效果。