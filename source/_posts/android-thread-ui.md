---
title: Android 子线程更新 UI
date: 2019-08-20 00:37:33
tags: View
categories: Android
---

# Android 子线程更新 UI

一般来说，不能在子线程更新 UI 控件，否则子线程和主线程同时更新 UI，可能导致 UI 被多个线程控制，显示异常。


## onCreate 更新 UI

下面这个例子，在 Activity 的 onCreate 可以更新 UI。

```java
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.worker_thread_activity);
        final TextView text = findViewById(R.id.text);
        new Thread(new Runnable() {
            @Override
            public void run() {
                text.setText("run in worker thread.");
            }
        }).start();
    }
```

但是，如果在子线程更新 TextView 之前，先 sleep 1 秒钟，就会抛出异常。

```java
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.worker_thread_activity);
        final TextView text = findViewById(R.id.text);
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                text.setText("run in worker thread.");
            }
        }).start();
    }
```

异常 log 如下，抛出了 CalledFromWrongThreadException 异常。异常明确说明了只有创建 View 的线程才可以更新这个 View。即只有主线程可以更新 View。

```java
    android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
        at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:8323)
        at android.view.ViewRootImpl.requestLayout(ViewRootImpl.java:1445)
        at android.view.View.requestLayout(View.java:24976)
        at android.view.View.requestLayout(View.java:24976)
        at android.view.View.requestLayout(View.java:24976)
        at android.view.View.requestLayout(View.java:24976)
        at android.view.View.requestLayout(View.java:24976)
        at android.widget.TextView.checkForRelayout(TextView.java:9643)
        at android.widget.TextView.setText(TextView.java:6231)
        at android.widget.TextView.setText(TextView.java:6059)
        at android.widget.TextView.setText(TextView.java:6011)
        at com.caoshen.androidsample.ui.WorkerThreadActivity$1.run(WorkerThreadActivity.java:27)
        at java.lang.Thread.run(Thread.java:919)
```

## onResume 更新 UI

下面这个例子，在 Activity 的 onResume 可以更新 UI。但是如果给子线程加上1秒钟延迟，同样抛出异常。

```java
    @Override
    protected void onResume() {
        super.onResume();
        final TextView text = findViewById(R.id.text);
        new Thread(new Runnable() {
            @Override
            public void run() {
                text.setText("run in worker thread.");
            }
        }).start();
    }
```

## checkThread

从前面的异常堆栈可以看出 setText 方法会调用 checkForRelayout，checkForRelayout 方法会调用 ViewRootImpl 的 requestLayout。在 requestLayout 会调用 checkThread 检查调用方法的线程是不是创建它的线程（主线程）。如果不是就会抛出 CalledFromWrongThreadException。

```java
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

```java
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

## onCreate/onResume 为什么可以更新

onCreate/onResume 回调开始时，View 的绘制流程还没有开始，也不会调用 requestLayout 重新布局，因此不会触发 checkThread 线程检查，从而不会抛出异常。但是，当延迟1秒钟后，View 的绘制流程已经完成，更新 UI 会触发 requestLayout 和线程检查，如果创建 View 和 更新 View 的不是同一个线程，就抛出 CalledFromWrongThreadException 异常。

查看 View 的 requestLayout 方法，可以发现当 mParent 不为空时，才会调用 mParent 的 requestLayout 方法。mParent.requestLayout 是一个 view 的递归调用，顶层的 ViewParent 就是 ViewRootImpl。但是 ViewRootImpl 的创建是在 onResume 之后，window addView 里面，之前 ViewRooltImpl 都是空，所以不会调用 ViewRootImpl 的 requestLayout，也就不会触发 checkThread 线程检查。所以 onCreate/onResume 可以更新 UI，直接调用 setText 方法。

```java
    public void requestLayout() {
        if (mMeasureCache != null) mMeasureCache.clear();

        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
            // Only trigger request-during-layout logic if this is the view requesting it,
            // not the views in its parent hierarchy
            ViewRootImpl viewRoot = getViewRootImpl();
            if (viewRoot != null && viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            mAttachInfo.mViewRequestingLayout = this;
        }

        mPrivateFlags |= PFLAG_FORCE_LAYOUT;
        mPrivateFlags |= PFLAG_INVALIDATED;

        if (mParent != null && !mParent.isLayoutRequested()) {
            mParent.requestLayout();
        }
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
            mAttachInfo.mViewRequestingLayout = null;
        }
    }
```