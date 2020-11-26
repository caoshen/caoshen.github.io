# RxJava 线程控制

RxJava 可以切换调度线程，控制每个操作在哪个线程执行。

## RxJava 内置的 Scheduler

如果我们不指定线程，默认是在调用 subscribe 方法的线程上进行回调的。如果想切换线程，就需要使用调度器（Scheduler）。RxJava 内置了如下 5 个 Scheduler。

1. Schedulers.immediate：直接在当前线程运行，它是 timeout、timeInterval 和 timestamp 操作符的默认调度器。

2. Schedulers.newThread: 总是启用新线程，在新线程执行操作。

3. Schedulers.io：IO （读写文件、数据库、网络信息交互）所使用的 Scheduler。io 内部实现是一个无数量上限的线程池，可以重用空闲的线程。因此多数情况下 io 比 newThread 更有效率。

4. Schedulers.computation: 计算所使用的 Scheduler，例如图形的计算。这个 Scheduler 使用固定线程池，大小为 CPU 核。不要把 IO 操作放到 computation 种，否则 IO 操作的等待时间会浪费 CPU 资源。它是 buffer、debounce、delay、interval、sample 和 skip 操作符的默认调度器。

5. Schedulers.trampoline: 当我们想在当前线程执行一个任务时，但不是立即执行时，可以使用 trampoline 将它入队。这个调度器会处理它的队列并且按照顺序运行。它是 repeat 和 retry 操作符默认的调度器。

## RxAndroid 的 Scheduler

RxAndroid 提供了主线程调度器 AndroidSchedulers.mainThread，可以指定操作在主线程。

## 控制线程

在 RxJava 中可以使用 subscribeOn 和 observeOn 来控制线程。subscribeOn 控制 observable 在哪个线程运行，observeOn 控制 observer 在哪个线程运行。

```java
    private void doSubscribleOn() {
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter emitter) throws Exception {
                Log.d(TAG, "Observable:" + Thread.currentThread().getName());
                emitter.onNext(1);
                emitter.onComplete();
            }
        }).subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<Integer>() {

                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.d(TAG, "Observer:" + Thread.currentThread().getName());
                    }
                });
    }
```

上面的代码 Observable 在子线程执行，Observer 在主线程执行。

输出结果如下：

```
Observable:RxNewThreadScheduler-1
Observer:main
```