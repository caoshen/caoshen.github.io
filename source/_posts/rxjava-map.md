---
title: RxJava 的 Map 变换过程解析
date: 2019-09-04 22:11:09
tags: Android
categories: Android
---

# RxJava 的 Map 变换过程解析

这里以 Map 操作符为例解析 RxJava 的变换过程。

## Map 操作

RxJava 中使用 Map 操作符的方式如下：

```java
    private void subscribeMap() {
        String start = "start:";
        Disposable disp = Observable
                .create((ObservableOnSubscribe<String>) emitter -> emitter.onNext("abc"))
                .map(s -> start + s)
                .subscribe(s -> Log.d(TAG, "accept:" + s));
    }
```

## create 方法解析

首先调用了 Observable 的 create 静态方法创建 Observable 对象。

create 代码如下：

```java
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

## map 方法解析

Observable 的 map 方法如下：

```java
    public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }
```

map 方法传入一个 Function 作为参数，返回一个 Observable 对象。map 方法被 final 修饰，说明它无法被子类改写。

与 create 方法类似，map 方法新建了 ObservableMap 类，并传给 RxJavaPlugins 的 onAssembly 方法。

之前说过，onAssembly 方法做了一个判断，如果有 onObservableAssembly 函数，先把 Observable 交给函数执行。否则直接返回 Observable。

经过 create 和 map 操作，得到一个 ObservableMap，它的构造方法如下：

```java
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }
```

可以看出 ObservableMap 继承了 AbstractObservableWithUpstream 类，也就是具有上游 Observable 的 Observable。对于 create 加上 map 的场景来说，上游 Observable 就是 ObservableCreate，也就是 ObservableMap 类里面的 source 参数。

因此，通过 Observable 的 create 方法，ObservableOnSubscribe 转换为 ObservableMap。

## subscribe 方法解析

在 Observable 的 create 和 map 之后，接下来调用了 Observable 的 subscribe 方法。

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

而当前的 Observable 就是 map 方法中的 ObservableMap。也就是说会调用 ObservableMap 的 subscribeActual 方法。

ObservableMap 的 subscribeActual 方法如下：

```java
    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
```

可以看出 ObservableMap 的 subscribeActual 方法调用了它的 source 的 subscribe 方法，并传入 MapObserver。

这里 source 就是传递给 ObservableMap 的 ObservableCreate 对象。

ObservableCreate 的 subscribe 方法会调用 emitter 的 onNext 方法将数据发射出去。然后在 ObservableMap 会接着调用 MapObserver 的 onNext 方法。

```java
    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        ...
        @Override
        public void onNext(T t) {
            ...
            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            }
            ...
            downstream.onNext(v);
        }
...
```

可以看出 MapObserver 的 onNext 方法会调用 mapper 映射函数的 apply，得到转换后的数据（t 转换为 v）。

这里的 downstream 就是下游订阅者 observer，也就是 subscribe 过程中的 LambdaObserver。

LambdaObserver 的 onNext 方法会调用示例代码中的 consumer，它调用了 accept 方法，将最终结果用 logcat 打印出来。

## 总结

1. 通过 Observable 的 map 方法，可以将 ObservableOnSubscribe 转换为 ObservableMap。

2. Observable 的 subscribe 方法会调用 ObservableMap 的 subscribeActual。在 subscribeActual 中会将 observer 转换为 MapObserver 并执行 subscribe 方法。

3. MapObserver 的 onNext 方法会执行 mapper 的 apply 方法，并将结果传递给下游 LambdaObserver，最后会执行 Consumer 的 accept。