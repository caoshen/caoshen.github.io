---
title: Android RxJava 基本用法
date: 2019-08-20 00:42:51
tags: RxJava
categories: Android
---

# Android RxJava 基本用法

RxJava 使用函数响应式编程方式，它可以简化项目，处理嵌套回调的异步事件。

## RxJava 依赖

这里以 RxJava 2.2.1 为例。在 build.gradle 添加依赖：

```
    // rxjava
    implementation "io.reactivex.rxjava2:rxjava:2.2.11"
    implementation 'io.reactivex.rxjava2:rxandroid:2.1.1'
```

## 创建观察者 Observer

新建一个 Observer，复写它的回调方法：onSubscribe、onNext、onError、onComplete

```java
Observer<String> observer = new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                Log.d(TAG, "onSubscribe:" + d);
            }

            @Override
            public void onNext(String s) {
                Log.d(TAG, "onNext:" + s);
            }

            @Override
            public void onError(Throwable e) {
                Log.d(TAG, "onError:" + e);
            }

            @Override
            public void onComplete() {
                Log.d(TAG, "onComplete:");
            }
        };
```

onSubscribe：订阅开始；

onNext：添加普通时间；

onError：异常事件；

onComplete：完结事件。

## 创建被观察者 Observable

创建 Observable 时可以使用 create、just 或者 from 方法。这里使用 just：

```java
        Observable<String> observable = Observable.just("杨影枫", "月眉儿");
```

以上代码相当于依次调用了 onNext("杨影枫") 和 onNext("月眉儿") ，最后调用 onComplete。

## 订阅 subscribe

使用 Observable 的 subscribe 方法可以触发订阅，代码如下：

```java
        observable.subscribe(observer);
```

运行结果 logcat 如下：

```
RxJavaActivity: onSubscribe:io.reactivex.internal.operators.observable.ObservableFromArray$FromArrayDisposable@1816f28
RxJavaActivity: onNext:杨影枫
RxJavaActivity: onNext:月眉儿
RxJavaActivity: onComplete:
```

## RxJava 不完整回调

RxJava 1.x 版本提供了 ActionX(X = 1~9、N) 来表示不同参数的回调。
在 RxJava 2 中使用 Consumer 代替 Action1，使用 BiConsumer 代替 Action2，使用 Consumer<Object[]> 代替 ActionN。

Action 代码如下：

```java
public interface Action {
    ...
    void run() throws Exception;
}
```

可以看出提供一个无参 run 方法。

Consumer 代码如下：

```java
public interface Consumer<T> {
    /**
     * Consume the given value.
     * @param t the value
     * @throws Exception on error
     */
    void accept(T t) throws Exception;
}
```

可以看出提供一个参数的 accept 方法。

BiConsumer 代码如下：

```java
public interface BiConsumer<T1, T2> {
...
    void accept(T1 t1, T2 t2) throws Exception;
}
```

可以看出提供 2 个参数的 accept 方法。

如果需要多个参数的方法，可以给 Consumer 类传入数组类型，即 Consumer<Object[]>。

有了 Action 和 Consumer，可以把之前的代码改写：

```java
    private void doRxJavaAction() {
        Consumer<String> nextConsumer = new Consumer<String>() {
            @Override
            public void accept(String s) {
                Log.d(TAG, "nextConsumer accept:" + s);
            }
        };

        Consumer<Throwable> errorConsumer = new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) {
                Log.d(TAG, "errorConsumer accept:" + throwable);
            }
        };

        Action completeAction = new Action() {
            @Override
            public void run() throws Exception {
                Log.d(TAG, "run:");
            }
        };
        Observable<String> observable = Observable.just("杨影枫", "月眉儿");

        observable.subscribe(nextConsumer, errorConsumer, completeAction);
    }
```

可以看出以上代码把回调方法拆分在 3 个回调对象中，然后传递给了 subscribe 方法。

subscribe 方法也可以接收 1 个或 2 个回调对象。

```java
        observable.subscribe(nextConsumer, errorConsumer);
```

这种方法写起来更加灵活，可以选择想要的回调方法。