---
title: Android 源码的模板方法模式
date: 2019-11-15 23:58:10
tags: 设计模式
categories: Android
---

# Android 源码的模板方法模式

## 模板方法模式介绍

在面向对象开发过程中，通常会遇到这样的一个问题，我们知道一个算法所需的关键步骤，并确定了这些步骤的执行顺序，但是某些步骤的具体实现是未知的，或者说某些步骤的实现是会随着环境的变化而改变的。

例如，执行程序的流程大致如下：

1. 检查代码的正确性
2. 链接相关的类库
3. 编译相关代码
4. 执行程序

对于不同的程序设计语言，上述 4 个步骤都是不一样的，但是它们的执行流程都是固定的，这类问题的解决方案就是模板方法模式。

## 模板方法模式的定义

定义一个操作中的算法的框架，而将一些步骤延迟到子类中，使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。

## Android 源码中的模板方法模式

在 Android 中，AsyncTask 是一个比较常用的类型，这个类就使用了模板方法模式。

AsyncTask 的整个执行过程其实是一个框架，具体的实现都需要子类来完成，而且它执行的算法框架是固定的，调用 execute 后会依次执行 onPreExecute、doInBackground、onPostExecute，当然也可以通过 onProgressUpdate 来更新进度。

AsyncTask 的 execute 方法如下：

```java
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```
executeOnExecutor 方法如下：

```java
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```

可以看到 execute 方法是一个 final 方法，它调用了 executeOnExecutor 方法。如果不是 Pending 状态会抛出依次，这也解释了为什么 AsyncTask 只能被执行一次，因为 AsyncTask 的 Running 和 Finished 状态都会抛出异常，因此每次使用 AsyncTask 时都需要重新创建一个对象。

### mWorker 和 mFuture

mWorker 只是实现了 Callable 接口，并添加了一个参数数组字段。

```java
    public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```

mWorker 的 call 方法会调用 doInBackground，并且在 finally 方法里面将 result 通过 postResult 方法传递出去。

mFuture 包装了 mWorker 对象，在这个 mFuture 对象的 run 函数中又会调用 mWorker 对象的 call 方法，在 call 方法中调用了 doInBackground 函数。因为 mFuture 提交给了线程池来执行，所以使得 doInBackground 执行在非 UI 线程。得到 doInBackground 的结果后，通过 postResult 传递结果给 UI 线程。

postResult 方法如下：

```java
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```

postResult 方法把一个消息（MESSAGE_POST_RESULT）发送给 Handler 执行。Handler 是 InternalHandler 类型。

```java
    private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```

当 InternalHandler 接到 MESSAGE_POST_RESULT 类型的消息时，就会调用 result.mTask.finish() 方法。

finish 方法如下：

```java
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```

AsyncTask 的 finish 方法又调用了 onPostExecute ，这个时候执行过程就完成了。

总之，execute 方法内部封装了 onPreExecute、doInBackground、onPostExecute 这个逻辑流程，用户可以根据自己的需求再覆写这几个方法，使得用户可以很方便地使用异步任务来完成耗时地操作以及更新UI，这其实就是通过线程池来执行耗时地任务，得到结果之后，通过 Handler 将结果传递到 UI 线程来执行。