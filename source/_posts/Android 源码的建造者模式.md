# Android 源码的建造者模式

建造者模式可以将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。

以 android 29 的代码为例，介绍 AlertDialog.Builder 和 ActivityStarter.Request。

## AlertDialog.Builder

Android 对话框 AlertDialog 的构建过程是一个典型的建造者模式，它的构造依赖内部类 Builder。

```java
    public static class Builder {
        @UnsupportedAppUsage
        private final AlertController.AlertParams P;
        ...
        public Builder(Context context) {
            this(context, resolveDialogTheme(context, Resources.ID_NULL));
        }
        ...
        public Builder setTitle(CharSequence title) {
            P.mTitle = title;
            return this;
        }
        ...
        public AlertDialog create() {
            // Context has already been wrapped with the appropriate theme.
            final AlertDialog dialog = new AlertDialog(P.mContext, 0, false);
            P.apply(dialog.mAlert);
            dialog.setCancelable(P.mCancelable);
            if (P.mCancelable) {
                dialog.setCanceledOnTouchOutside(true);
            }
            dialog.setOnCancelListener(P.mOnCancelListener);
            dialog.setOnDismissListener(P.mOnDismissListener);
            if (P.mOnKeyListener != null) {
                dialog.setOnKeyListener(P.mOnKeyListener);
            }
            return dialog;
        }
...
        public AlertDialog show() {
            final AlertDialog dialog = create();
            dialog.show();
            return dialog;
        }
    }
```

可以看出调用 setTitle 会把 title 赋值给  AlertController.AlertParams P，在 create 的时候会使用 P.apply 传递参数到 dialog 的 mAlert 中，最后返回构建好的 dialog。


```java
        public void apply(AlertController dialog) {
            if (mCustomTitleView != null) {
                dialog.setCustomTitle(mCustomTitleView);
            } else {
                if (mTitle != null) {
                    dialog.setTitle(mTitle);
                }
                if (mIcon != null) {
                    dialog.setIcon(mIcon);
                }
                ...
        }

```

AlertParams 的 apply 方法会把参数传递给 AlertController。

Dialog 的 show 方法如下：

```java
    public void show() {
        ...
        if (!mCreated) {
            dispatchOnCreate(null);
        }
        ...
        onStart();
        mDecor = mWindow.getDecorView();
        ...
        mWindowManager.addView(mDecor, l);
        ...
        mShowing = true;

        sendShowMessage();
    }
```

可以看出 show 方法回调了 create、start 的生命周期方法，最后通过 mWindowManager.addView 显示出来。

## ActivityStarter.Request

查看 ActivityTaskManagerService 的 startActivityAsUser 方法。

```java
    int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        ...
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();

    }
```

startActivityAsUser 构造了 ActivityStarter 并调用它的 execute 启动 Activity。


```java
    ActivityStarter setCaller(IApplicationThread caller) {
        mRequest.caller = caller;
        return this;
    }
```

ActivityStarter 的 setCaller 方法把 caller 赋值给 mRequest，然后返回它自己。

```java
    ActivityStarter setMayWait(int userId) {
        mRequest.mayWait = true;
        mRequest.userId = userId;

        return this;
    }
```

ActivityStarter 的 setMayWait 方法把 mayWait 赋值为 true，也就是说要等待启动 Activity 的结果，然后传递 userId。

ActivityStarter 的 execute 方法如下：

```java
int execute() {
        try {
            // TODO(b/64750076): Look into passing request directly to these methods to allow
            // for transactional diffs and preprocessing.
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                        mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            }
        } finally {
            onExecutionComplete();
        }
    }
```

可以看出 execute 方法根据 mRequest.mayWait 值，判断是执行 startActivityMayWait 还是 startActivity，最后把 mRequest 的每个属性传递给了这两个方法，完成 Activity 的启动。