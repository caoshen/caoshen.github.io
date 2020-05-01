# Activity 难点

记录 Activity 相关的问题。

## setResult 和 finish 的关系

假设 Activity A 跳转到 Activity B，B 如果想回传一个 intent 给 A，setResult 和 finish 的执行顺序应该是怎样的？

应该先 setResult，然后再 finish。

```java
    public final void setResult(int resultCode, Intent data) {
        synchronized (this) {
            mResultCode = resultCode;
            mResultData = data;
        }
    }
```

可以看出，setResult 的作用是将 resultCode 和 data 赋值给 Activity 的 mResultCode 和 mResultData 两个字段。

```java
    public void finish() {
        finish(DONT_FINISH_TASK_WITH_ACTIVITY);
    }
```

可以看出，finish 调用的是传递了 DONT_FINISH_TASK_WITH_ACTIVITY 参数的 finish 方法。

```java
private void finish(int finishTask) {
        if (mParent == null) {
            int resultCode;
            Intent resultData;
            synchronized (this) {
                resultCode = mResultCode;
                resultData = mResultData;
            }
            if (false) Log.v(TAG, "Finishing self: token=" + mToken);
            try {
                if (resultData != null) {
                    resultData.prepareToLeaveProcess(this);
                }
                if (ActivityManager.getService()
                        .finishActivity(mToken, resultCode, resultData, finishTask)) {
                    mFinished = true;
                }
            } catch (RemoteException e) {
                // Empty
            }
        } else {
            mParent.finishFromChild(this);
        }

        ...
    }
```

可以看出，在 finish 方法里面会保存 mResultCode 和 mResultData 作为临时变量，然后作为参数调用 AMS 的 finishActivity 方法。如果 finish 在 setResult 之前，那么 setResult 的 resultCode 和 resultData 就不会传递给 AMS，也就不会传递给 Activity A 的 onActivityResult 方法。

此时 A 和 B 的生命周期回调方法：

B -> onPause

A -> onRestart

A -> onStart

A -> onResume

B -> onStop

B -> onDestory


## onSaveInstanceState 和 onRestoreInstanceState

如果用户按 back 键主动退出应用或者程序调用 finish 方法时，onSaveInstanceState() 不会被调用。如果是系统出于某种原因需要销毁 Activity 时，销毁之前系统会调用 onSaveInstanceState() 方法。

生命周期变化如下：

Resumed -> onSaveInstanceState -> Destroyed

onCreate -> created -> onRestoreInstanceState -> Resumed

```java
    @Override
    protected void onSaveInstanceState(Bundle outState) {
        // 保存自定义的状态
        outState.putString(KEY, value);
        // 系统保存 view 里面的状态
        super.onSaveInstanceState(outState);
    }

    @Override
    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        // 系统恢复 view 里面的状态
        super.onRestoreInstanceState(savedInstanceState);
        // 恢复自定义的状态
        String value = savedInstanceState.getString(KEY);
    }
```

## onNewIntent

如果当前 Activity 处于栈顶，同时它的启动模式是 singleTop，或者启动它的 intent 具有 FLAG_ACTIVITY_SINGLE_TOP，那么启动它时不会重走 onCreate，而是调用 onNewIntent 方法。

onPause() -> onNewIntent() -> onResume()

如果当前 Activity 在栈中，同时它的启动模式是 singleTask 或者 singleInstance，那么也会调用 onNewIntent 方法。

onNewIntent() -> onRestart() -> onStart() -> onResume()

有一点需要注意，onNewIntent 回调之后，需要把新的 intent 赋值给 activity 的 mIntent 变量，否则每次 get 到的 intent 都是老的 intent。

```java
    /** Return the intent that started this activity. */
    public Intent getIntent() {
        return mIntent;
    }

    /**
     * Change the intent returned by {@link #getIntent}.  This holds a
     * reference to the given intent; it does not copy it.  Often used in
     * conjunction with {@link #onNewIntent}.
     *
     * @param newIntent The new Intent object to return from getIntent
     *
     * @see #getIntent
     * @see #onNewIntent
     */
    public void setIntent(Intent newIntent) {
        mIntent = newIntent;
    }
```

从 Activity 的代码可以看出，getIntent 和 setIntent 只是单纯的设置和返回 mIntent 变量。所以必须在 onNewIntent 的时候调用 setIntent 方法。

```java
    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        setIntent(intent);
    }
```

## onConfigurationChanged

系统配置变化，比如屏幕旋转、分屏时，会导致 Activity 销毁重建，重走生命周期。如果在 AndroidManifest 里面配置了 Activity 的 configChanges 属性，那么 configChanges 属性里面对应的配置更改了，不会导致 Activity 销毁重建，而是执行 Activity 的 onConfigurationChanged 方法。

```xml
android:configChanges="orienetation | screenSize"
```

orienetation | screenSize 保证了当屏幕旋转，屏幕布局发生了变化时，Activity 不会销毁。

## Activity 生命周期

正常情况下的生命周期：

onCreate -> onStart -> onResume -> onPause -> onStop -> onDestroy

有时候，当 Activity 被重新启动时，会执行 onRestart 方法。

onRestart -> onStart -> onResume -> onPause -> onStop -> onDestroy

## Activity 启动模式

Activity 的启动模式有 4 种：

1. standard 标准启动模式。

2. singleTop 栈顶复用模式。

3. singleTask 栈内复用模式。

4. singleInstance 单实例模式。

## Activity IntentFilter

一个 Activity 可以有多个 IntentFilter。隐式启动 Activity 时，只要有一个 IntentFilter 匹配，那么 Activity 就能被启动。

一个 IntentFilter 里面可以配置多个 action，多个 category，多个 data。

### action

如果启动的 intent 的 action 和 intentFilter 中的一个 action 相同，那么就算 action 匹配成功。如果 intent 的 action 没有和 intentFilter 中的任何一个 action 相同，那么匹配失败。如果 intent 没有 action，也会匹配失败。

### category

如果启动的 intent 有多个 category，那么每个 category 都在 intentFilter 的 category 列表中，才算 category 匹配成功。如果 intent 不设置 category，那么不会去匹配 intentFilter 的 category。

### data

如果启动的 intent 有 data，那么只要 data 在 intentFilter 的 data 列表中，就算 data 匹配成功。data 的匹配过程和 action 类似。如果 intentFilter 定义了 data，但是启动的 intent 没有 data，那么也会匹配失败。
