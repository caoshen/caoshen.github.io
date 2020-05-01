# RxJava 订阅过程解析

这里以 RxJva 2.2.11 为例， 从 RxJava 的订阅过程解析 RxJava 源码。

## RxJava 的订阅过程

RxJava 的基本订阅流程如下：

```java
    private void subcribeSource() {
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                emitter.onNext(1);
            }
        }).subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer o) throws Exception {
                
            }
        });
    }
```

## create

首先调用了 Observable 的静态 create 方法创建 Observable 对象。

create 代码如下：

```java
    @CheckReturnValue
    @NonNull
    @SchedulerSupport(SchedulerSupport.NONE)
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }
```

直接调用了 RxJavaPlugins 的 onAssembly，同时创建一个 ObservableCreate 作为参数。

ObservableCreate 类如下：

```java
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }
    ...
}
```

ObservableCreate 继承了 Observable 类，将构造方法的 ObservableOnSubscribe 作为 final 的成员变量。

接下来看 RxJavaPlugins 的 onAssembly 方法：

```java
    public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
        Function<? super Observable, ? extends Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }
```

onAssembly 方法做了一个判断，如果有 onObservableAssembly 函数，先把 Observable 交给函数执行。否则直接返回 Observable。

这样的用处是在 Observable 被订阅之前加上一些自定义的操作，而又不会影响原有的订阅流程。

通过 Observable 的 create 方法，ObservableOnSubscribe 转换为 ObservableCreate。

## subscribe

在 Observable 的 create 之后，接下来调用了 Observable 的 subscribe 方法。

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

首先对每个参数进行判空操作，然后用 4 个参数构造了LambdaObserver，LambdaObserver 实现了 Observer<T>, Disposable 等接口。最后调用了 subscribe 方法。 

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

之前分析得到 create 方法最后会返回 ObservableCreate。

ObservableCreate 的 subscribeActual 方法如下：

```java
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
```

可以看出，observer 被封装成 CreateEmitter。CreateEmitter 给 observer 的每个回调方法增加了 dispose 和错误处理。

然后调用了 observer 的 onSubscribe 回调，并传入 CreateEmitter。

最后调用 source 的 subscribe 方法，即第一步 create 传入的 ObservableOnSubscribe 对象的 subscribe 方法。

在示例代码中的 subscribe 方法调用了 emitter.onNext(1)。因此 parent (即 CreateEmitter 封装后的 observer) 会调用它的 onNext 方法，并传入参数 1。

因此会调用 CreateEmitter 的 onNext:

```java
        @Override
        public void onNext(T t) {
            if (t == null) {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }
```

可以看出会把 observable 发射的参数 1 传递给 Observer 的 onNext。 

而这个 observable 就是上面提到的 LambdaObserver。

LambdaObserver 的 onNext 方法如下：

```java
    @Override
    public void onNext(T t) {
        if (!isDisposed()) {
            try {
                onNext.accept(t);
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                get().dispose();
                onError(e);
            }
        }
    }
```

可以看出，它会调用自己的 onNext 成员变量的 accept 方法。而这个 onNext 成员变量就是示例代码中传入的 Consumer。

因此最终会调用到 Consumer 的 accept 方法。即订阅者（Oberver）收到了被订阅者（Observable）发射的参数 1。

## 总结

1. 通过 Observable 的 create 方法，可以将 ObservableOnSubscribe 转换为 ObservableCreate。

2. Observable 的 subscribe 方法会调用 ObservableCreate 的 subscribeActual。在 subscribeActual 中会将 observer 转换为 CreateEmitter 并执行 subscribe 方法。

3. CreateEmitter 的 onNext 等方法最终会执行 observer 的 onNext 方法，也就是 Consumer 的 accept。
