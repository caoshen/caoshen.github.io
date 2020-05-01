# RxJava 线程切换过程解析

RxJava 可以配合 RxAndroid 使用 Schedulers 和 AndroidSchedulers 完成线程的切换。

## 线程切换

```java
    private void subscribeOn() {
        Disposable disp = Observable
                .create(emitter -> {
                    emitter.onNext(1);
                    emitter.onComplete();
                })
                .subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(i -> Log.d(TAG, "accept:" + i));
    }
```

以上代码首先调用 Observable.create 方法构造 Observable，然后调用 subscribeOn 方法在子线程执行，接着调用 observeOn 方法切换到主线程执行。最后调用 subscribe 方法打印订阅到的值。

## subscribeOn 流程解析

subscribeOn 方法如下：

```java
    public final Observable<T> subscribeOn(Scheduler scheduler) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
    }
```

和 Observable 的其他方法类似，subscribeOn 方法构造了 ObservableSubscribeOn，然后由 RxJavaPlugins 的 onAssembly 调用。

ObservableSubscribeOn 代码如下：

```java
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> observer) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

        observer.onSubscribe(parent);

        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }
    ...
}
```

ObservableSubscribeOn 的构造方法传入了 source 和 scheduler，分别是源 ObservableSource 和 调度器 scheduler。这里的 ObservableSource 是上一步 create 得到的 ObservableCreate 类，scheduler 是 Schedulers.newThread() 得到的 NewThreadScheduler，本质上是一个线程的 ScheduledThreadPoolExecutor。

ObservableSubscribeOn 的 subscribeActual 将 observer 封装成 SubscribeOnObserver。然后调用了 observer 的 onSubscribe 方法和 parent 的 setDisposable 方法。

这里主要看一下 parent.setDisposable 的参数 scheduler.scheduleDirect(new SubscribeTask(parent))。

```java
    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }
```

SubscribeTask 实现了 Runnable 接口，在 run 方法调用 source.subscribe(parent)，也就是把上一步 create 的操作放到 Runnable 执行。

scheduler.scheduleDirect 对于 Schedulers.newThread() 调度器来说，就是 NewThreadWorker 的 scheduleDirect 方法。

```java
    public Disposable scheduleDirect(final Runnable run, long delayTime, TimeUnit unit) {
        ScheduledDirectTask task = new ScheduledDirectTask(RxJavaPlugins.onSchedule(run));
        ...
            if (delayTime <= 0L) {
                f = executor.submit(task);
            } else {
                f = executor.schedule(task, delayTime, unit);
            }
            task.setFuture(f);
            return task;
        ...
    }
```

之前说过 scheduler 是 Schedulers.newThread() 得到的 NewThreadScheduler，本质上是一个线程的 ScheduledThreadPoolExecutor。这里的 scheduleDirect 方法本质上调用了 submit 方法（如果 delay > 0，有延迟，则是 schedule 方法）。因此我们通过第一步 create 的 emitter.onNext(1) 发射数据的操作是在一个单线程池执行的。

## observeOn 流程解析

observeOn 方法如下：

```java
    public final Observable<T> observeOn(Scheduler scheduler) {
        return observeOn(scheduler, false, bufferSize());
    }
```

可以看出 observeOn 调用了 3 参数的 observeOn。

3 参数的 observeOn 方法如下：

```java
    public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        ObjectHelper.verifyPositive(bufferSize, "bufferSize");
        return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
    }
```

observeOn 方法构造了 ObservableObserveOn，然后由 RxJavaPlugins 的 onAssembly 调用。

```java
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        if (scheduler instanceof TrampolineScheduler) {
            source.subscribe(observer);
        } else {
            Scheduler.Worker w = scheduler.createWorker();

            source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        }
    }
```

可以看出 subscribeActual 方法调用了 scheduler 的 createWorker，然后创建一个 ObserveOnObserver 作为真正的 observer 交给 source subscribe。

这里的 worker 就是 scheduler 创建的，而 scheduler 传入的是一个 AndroidSchedulers.mainThread() 得到的主线程调度器。最终会返回 AndroidSchedulers 的 MainHolder.DEFAULT 对象。

MainHolder.DEFAULT 代码如下：

```java
/** Android-specific Schedulers. */
public final class AndroidSchedulers {

    private static final class MainHolder {
        static final Scheduler DEFAULT
            = new HandlerScheduler(new Handler(Looper.getMainLooper()), false);
    }
    ...
```

可以看出 AndroidSchedulers.mainThread() 调度器本质上就是一个主线程的 Handler（由主线程的 Looper 构造的 Handler）。所有的订阅者事件都会被封装成主线程消息发送到主线程执行。

## subscribe 流程解析

在 Observable 的 create、subscribeOn、observeOn 方法之后，接下来调用了 Observable 的 subscribe 方法。

subcribe 方法有很多重载方法。这里以一个 Consumer 的方法为例。

```java
public final Disposable subscribe(Consumer<? super T> onNext) {
    return subscribe(onNext, Functions.ON_ERROR_MISSING, Functions.EMPTY_ACTION, Functions.emptyConsumer());
}
```

可以看出它调用了 4 参数的 subscribe 方法，返回一个 Disposable 对象。

4 参数的 subscribe 方法的代码如下：

```java
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
        Action onComplete, Consumer<? super Disposable> onSubscribe) {
    ObjectHelper.requireNonNull(onNext, "onNext is null");
    ObjectHelper.requireNonNull(onError, "onError is null");
    ObjectHelper.requireNonNull(onComplete, "onComplete is null");
    ObjectHelper.requireNonNull(onSubscribe, "onSubscribe is null");

    LambdaObserver<T> ls = new LambdaObserver<T>(onNext, onError, onComplete, onSubscribe);

    subscribe(ls);

    return ls;
}
```

首先对每个参数进行判空操作，然后用 4 个参数构造了LambdaObserver，LambdaObserver 实现了 Observer, Disposable 等接口。最后调用了 subscribe 方法。

subscribe 方法如下：

```java
public final void subscribe(Observer<? super T> observer) {
    ....
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);

        ObjectHelper.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook returned a null Observer. Please change the handler provided to RxJavaPlugins.setOnObservableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");

        subscribeActual(observer);
    }
    ...
}
```

可以看出实际上调用了当前 Observable 的 subscribeActual 方法。

而当前的 Observable 其实就是封装了 create、subscribeOn、observeOn 的 ObservableObserveOn 对象。

从 observeOn 流程解析可以看出，ObservableObserveOn 的 subscribeActual 方法构造了一个 Scheduler.Worker，然后将原始 observer 和 worker 封装为新的 ObserveOnObserver，交给 ObservableObserveOn 的上一级 source，也就是 ObservableSubscribeOn 的 subscribe 方法。

这里我们还是看下 ObservableObserveOn 的内部类 ObserveOnObserver。ObserveOnObserver 实现了 Runnable 接口，它的 schedule 方法就是把自己扔到 worker 中执行。

ObserveOnObserver 的 run 方法如下：

```java
        @Override
        public void run() {
            if (outputFused) {
                drainFused();
            } else {
                drainNormal();
            }
        }
```

正常情况下 outputFused 为 false，会调用 drainNormal。

drainNormal 方法如下：

```java
        void drainNormal() {
            ...
            final Observer<? super T> a = downstream;

            for (;;) {
                ...
                        v = q.poll();
                    ...
                    a.onNext(v);
                }
...
        }
```

可以看出 drainNormal 会调用 downstream 的 onNext 方法，就是下游 observer 的 onNext 方法。下游 observer 其实就是 LambdaObserver ，也就是说会调用 LambdaObserver 的 onNext，最后会 LambdaObserver 的 onNext 方法会调用示例代码中的 consumer，它调用了 accept 方法，将最终结果用 logcat 打印出来。

## 总结

1. 通过 Observable 的 create 方法，可以将 ObservableOnSubscribe 转换为 ObservableCreate。

2. 通过 Observable 的 subscribeOn 方法，可以将 ObservableCreate 转换为 ObservableSubscribeOn，第一步的发射数据操作会被封装成 Runnable 放入单线程池执行。

3. 通过 Observable 的 observeOn 方法，可以将 ObservableSubscribeOn 转换为 ObservableObserveOn。传入的 AndroidSchedulers.mainThread() 调度器本质上就是一个主线程的 Handler。所有的订阅者事件都会被封装成主线程消息发送到主线程执行。

4. Observable 的 subscribe 方法会调用 ObservableObserveOn 的 subscribeActual，然后在 drainNormal 方法中会调用到 observer 的 onNext 方法，执行真正的订阅者操作。


