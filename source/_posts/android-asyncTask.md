---
title: AsyncTask 异步任务
date: 2019-08-20 00:38:57
tags: Android
categories: Android
---

# AsyncTask 异步任务

AsyncTask 是 Android 的异步任务类，它本质上是对 Thread 和 Handler 的封装，达到工作线程执行异步任务，执行完毕后切换到主线程的功能。

## AsyncTask 的执行顺序

以 SDK 28 为例，默认情况下，调用 execute 方法，AsyncTask 的任务都是串行的。比如有 10 个 AsyncTask 同时开始执行，实际上当前一个 AsyncTask 执行完毕后，下一个AsyncTask 才会真正开始执行。

```java
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```

以上代码可以看出，默认使用了 sDefaultExecutor 来执行异步任务。

```java
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```

```java
    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```

```java
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```

sDefaultExecutor 是一个 SerialExecutor 的单例。SerialExecutor 内部有一个双端队列。每次执行的时候，先入队一个封装过的 runnable，如果当前没有任务在执行（mActive == null），就从队列出队一个 runnable，然后交给线程池执行。封装的 runnable 是 try-finally 结构，也就是说不管当前任务是否在执行过程中抛出异常，都会接着执行下一个任务，即从队列出队，交给线程池执行。这样的队列执行方式保证了 asyncTask 都是在队列中串行执行。

由于 AsyncTask 提供了另外一个方法 executeOnExecutor，所以可以传递一个 executor 参数。如果传递的 executor 是多线程池，那么也能让 AsyncTask 并行执行。

## execute 方法调用次数

execute 方法只能调用一次。一旦 asyncTask 执行完毕，它的状态会变成 FINISHED。executeOnExecutor 方法会判断当前状态，如果是 FINISHED 或者 RUNNING，直接抛出异常。所以不可以对同一个 asyncTask 多次调用 execute 方法，多次调用 executeOnExecutor 方法也不行。

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

## AsyncTask 所在的线程

正常来说，都是在主线程创建和使用 AsyncTask。但是如果把 AsyncTask 放入子线程创建，其实也是可以的。假如已经在子线程了，就可以直接执行异步任务，没有必要再用 AsyncTask。

```java
/**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     */
    public AsyncTask() {
        this((Looper) null);
    }

    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     *
     * @hide
     */
    public AsyncTask(@Nullable Handler handler) {
        this(handler != null ? handler.getLooper() : null);
    }

    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     *
     * @hide
     */
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

从以上代码可以看出，如果使用默认的构造方法，传入一个空 Looper，那么会使用 getMainHandler 方法获取主线程的 Handler。实际上使用主线程的 looper 构造一个 handler，用来在完成异步任务和更新异步任务的进度时，切换到主线程。这一步是通过消息机制完成的。

```java
    private static Handler getMainHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;
        }
    }
```

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

通过 InternalHandler 的 handleMessage 方法，可以明确看出，任务执行完毕的消息和更新任务的消息都是在主线程执行。因为消息对应的 looper 是构造 InternalHandler 时传入的主线程的 looper。

### doInBackground 所在线程

doInBackground 执行在 AsyncTask 的线程池 THREAD_POOL_EXECUTOR 中，也就是工作线程。

### onPreExecute 所在线程

onPreExecute 执行在 AsyncTask 所在线程，通常是主线程，但是如果把 AsyncTask 放入子线程中，就在子线程。

### onPostExecute 所在线程

onPostExecute 执行在主线程。

### onProgressUpdate 所在线程

onProgressUpdate 执行在主线程。

### onCancelled 所在线程

onCancelled 执行在主线程。

### 自定义 AsyncTask

自定义一个 AsyncTask，然后放在子线程执行。

```java
@Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        new Thread(new Runnable() {
            @Override
            public void run() {
                SampleAsyncTask sampleAsyncTask = new SampleAsyncTask();
                sampleAsyncTask.execute();
                // sampleAsyncTask.cancel(true);
            }
        }).start();
    }
```

```java
    private static class SampleAsyncTask extends AsyncTask<Void, Void, Void> {

        private static final String TAG = "SampleAsyncTask";

        @Override
        protected Void doInBackground(Void... voids) {
            Log.e(TAG, "doInBackground:" + Thread.currentThread().getName());
            publishProgress();
            return null;
        }

        @Override
        protected void onPreExecute() {
            super.onPreExecute();
            Log.e(TAG, "onPreExecute:" + Thread.currentThread().getName());
        }

        @Override
        protected void onPostExecute(Void aVoid) {
            super.onPostExecute(aVoid);
            Log.e(TAG, "onPostExecute:" + Thread.currentThread().getName());
        }

        @Override
        protected void onProgressUpdate(Void... values) {
            super.onProgressUpdate(values);
            Log.e(TAG, "onProgressUpdate:" + Thread.currentThread().getName());
        }

        @Override
        protected void onCancelled() {
            super.onCancelled();
            Log.e(TAG, "onCancelled:" + Thread.currentThread().getName());
        }

    }
```

执行结果如下：

```
2019-07-14 21:08:21.362 27817-27892/com.caoshen.androidsample E/SampleAsyncTask: onPreExecute:Thread-5
2019-07-14 21:08:21.374 27817-27893/com.caoshen.androidsample E/SampleAsyncTask: doInBackground:AsyncTask #1
2019-07-14 21:08:21.518 27817-27817/com.caoshen.androidsample E/SampleAsyncTask: onProgressUpdate:main
2019-07-14 21:08:21.518 27817-27817/com.caoshen.androidsample E/SampleAsyncTask: onPostExecute:main
```

调用 cancel 方法：

```
2019-07-14 21:09:35.084 28942-29022/com.caoshen.androidsample E/SampleAsyncTask: onPreExecute:Thread-5
2019-07-14 21:09:35.085 28942-29023/com.caoshen.androidsample E/SampleAsyncTask: doInBackground:AsyncTask #1
2019-07-14 21:09:35.245 28942-28942/com.caoshen.androidsample E/SampleAsyncTask: onProgressUpdate:main
2019-07-14 21:09:35.245 28942-28942/com.caoshen.androidsample E/SampleAsyncTask: onCancelled:main
```

可以看出 onPreExecute 所在线程和 AsyncTask 所在线程有关，如果 AsyncTask 在主线程执行，onPreExecute 就是在主线程，如果 AsyncTask 在子线程执行，onPreExecute 就是在子线程。

doInBackground 始终在 AsyncTask 的线程池中执行。这是因为 doInBackground 在封装的 Callable 中并交给线程池执行。

onPostExecute、onProgressUpdate、onCancelled 执行在主线程，因为这三个方法通过消息机制由 InternalHandler 发送到主线程执行，InternalHandler 对应的 looper 也是主线程的 looper。