---
title: RxJava 操作符介绍
date: 2019-08-26 23:54:30
tags: Android
categories: Android
---

# RxJava 操作符介绍

RxJava 操作符的类型可以分为：

1. 创建操作符
2. 变换操作符
3. 过滤操作符
4. 组合操作符
5. 错误处理操作符
6. 辅助操作符
7. 条件布尔操作符
8. 算术聚合操作符
9. 连接操作符

这些操作符类型下面由很多操作符，每个操作符可能还有很多变体。

## 创建操作符

创建操作符有 create、just、fromArray，以及 defer、range、interval、start、repeat 和 timer 等。

### create

```java
        Observable<String> observable1 = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("abc");
                emitter.onNext("def");
                emitter.onComplete();
            }
        });
```

### just

```java
Observable<String> observable = Observable.just("abc", "def");
```

### fromArray

```java
        String[] words = {"abc", "def"};
        Observable<String> observable2 = Observable.fromArray(words);
```

### interval

```java
    private void doInterval() {
        Disposable subscribe = Observable.interval(3, TimeUnit.SECONDS)
                .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(Long aLong) throws Exception {
                        Log.d(TAG, "accept:" + aLong);
                    }
                });
    }
```

每隔 3 秒会调用 accept 方法。

### range

```java
    private void doRange() {
        Disposable subscribe = Observable.range(0, 5)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.d(TAG, "accept:" + integer);
                    }
                });
    }
```

从 0 开始调用 call 5 次。start = 0, count = 5。

### repeat

```java
    private void doRepeat() {
        Disposable subscribe = Observable.range(0, 5)
                .repeat(2)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.d(TAG, "accept:" + integer);
                    }
                });
    }
```

从 0 开始调用 call 5 次，总共重复 2 遍。times = 2。

## 变换操作符

变换操作符包括 map、flatMap、concatMap、switchMap、flatMapIterable、buffer、groupBy、cast、window、scan 等。这里介绍其中几种。

### map

map 操作符可以通过一个 Function 将一个 Observable 转换为另一个 Observable。

```java
    private void doMap() {
        String start = "abc";
        Disposable subscribe = Observable.just("def").map(new Function<String, String>() {
            @Override
            public String apply(String s) throws Exception {
                return start + s;
            }
        }).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.d(TAG, "accept:" + s);
            }
        });
    }
```

### flatMap 和 cast

flatMap 可以将 Observable 集合转换成一个单独的 Observable。flatMap 的合并允许交叉，可能会交错的发送事件，最后的顺序可能不是原始 Observable 发送时的顺序。

cast 可以将 Observable 转换为指定类型，比如 String 类型。

```java
    private void doFlatMapAndCast() {
        String start = "abc";
        List<String> strList = new ArrayList<>();
        strList.add("abc123");
        strList.add("abc124");
        strList.add("abc125");
        strList.add("abc126");
        Disposable subscribe = Observable.fromIterable(strList).flatMap(new Function<String, ObservableSource<?>>() {
            @Override
            public ObservableSource<?> apply(String s) throws Exception {
                return Observable.just(start + s);
            }
        }).cast(String.class).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.d(TAG, "doFlatMapAndCast:" + s);
            }
        });
    }
```

### concatMap

concatMap 操作符和 flatMap 功能类似，不同的是它解决了 flatMap 的交叉问题，能把发射的值连续在一起。

```java
    private void doConcatMap() {
        String start = "def-";
        List<String> strList = new ArrayList<>();
        strList.add("abc123");
        strList.add("abc124");
        strList.add("abc125");
        strList.add("abc126");
        Disposable subscribe = Observable.fromIterable(strList).concatMap(new Function<String, ObservableSource<?>>() {
            @Override
            public ObservableSource<?> apply(String s) throws Exception {
                Log.d(TAG, "doConcatMap: apply:" + s);
                return Observable.just(start + s);
            }
        }).cast(String.class).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.d(TAG, "doConcatMap:" + s);
            }
        });
    }
```

### buffer

buffer 操作符可以将一组值一同发送，而不是一个一个发送。

```java
    private void doBuffer() {
        Observable.just(1, 2, 3, 4, 5, 6)
                .buffer(3)
                .subscribe(new Consumer<List<Integer>>() {
                    @Override
                    public void accept(List<Integer> integers) throws Exception {
                        Log.d(TAG, "doBuffer: " + integers);
                        Log.d(TAG, "------------");
                    }
                });
    }
```

日志如下：

```
D/RxJavaActivity: doBuffer: [1, 2, 3]
D/RxJavaActivity: ------------
D/RxJavaActivity: doBuffer: [4, 5, 6]
D/RxJavaActivity: ------------
```

### groupBy

groupBy 可以将 Observable 根据某个函数来分组。

```java
    private void doGroupBy() {
        Observable<GroupedObservable<Character, String>> observable = Observable.just("apple", "pear", "orange", "pen", "oreo", "pie")
                .groupBy(new Function<String, Character>() {
                    @Override
                    public Character apply(String s) throws Exception {
                        return s.charAt(0);
                    }
                });
        Observable.concat(observable).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.d(TAG, "doGroupBy:" + s);
            }
        });

    }
```

上面的方法将单词按照单词的第一个字母分组。

日志如下：

```
D/RxJavaActivity: doGroupBy:apple
D/RxJavaActivity: doGroupBy:pear
D/RxJavaActivity: doGroupBy:pen
D/RxJavaActivity: doGroupBy:pie
D/RxJavaActivity: doGroupBy:orange
D/RxJavaActivity: doGroupBy:oreo
```

## 过滤操作符

过滤操作符有 filter、elementAt、distinct、skip、take、ignoreElements、throttleFirst 和 throttleWithTimeOut 等。

### filter 

filter 可以对 observable 进行过滤。

```java
    private void doFilter() {
        Observable.just(1, 2, 3, 4, 5).filter(new Predicate<Integer>() {
            @Override
            public boolean test(Integer integer) throws Exception {
                return integer > 2;
            }
        }).subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                Log.d(TAG, "filter:" + integer);
            }
        });
    }
```

以上代码可以过滤大于 2 的整数，即 3、4、5。

### elementAt

elementAt 可以用来返回指定位置的数据。和它类似的有 elementAtOrDefault(int, T)，其可以允许默认值。

```java
    private void doElementAt() {
        Observable.just(1, 2, 3, 4, 5).elementAt(2)
                .subscribe((integer) -> {
                    Log.d(TAG, "elementAt:" + integer);
                });
    }
```
以上代码会打印 3。

### distinct

distinct 可以用来去重，只允许没有发射过的数据项通过。

```java
    private void doDistinct() {
        Observable.just(1, 2, 2, 3, 4, 4, 5).distinct()
                .subscribe(integer -> {
                    Log.d(TAG, "distinct:" + integer);
                });
    }
```
以上代码会打印12345，把重复的数字去掉。

### skip

skip 可以跳过前 N 项。

```java
    private void doSkip() {
        Observable.just(1, 2, 3, 4, 5)
                .skip(3)
                .subscribe(integer -> {
                    Log.d(TAG, "skip:" + integer);
                });
    }
```
以上代码会打印 45。

### take

take 可以只取前 N 项。

```java
    private void doTake() {
        Observable.just(1, 2, 3, 4, 5)
                .take(3)
                .subscribe(integer -> Log.d(TAG, "take:" + integer));
    }
```
以上代码会打印 123。

### ignoreElements

ignoreElements 会忽略 observable 的元素。

```java
    private void doIgnoreElements() {
        Observable.just(1, 2, 3, 4, 5)
                .ignoreElements()
                .subscribe(() -> Log.d(TAG, "onCompleted"));
    }
```
以上代码会打印 onCompleted。

### throttleFirst

throttleFirst 会定期发射指定时间段里的 Observable 发射的第一个数据。

```java
    private void doThrottleFirst() {
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                for (int i = 0; i < 10; ++i) {
                    emitter.onNext(i);
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                emitter.onComplete();
            }
        })
                .throttleFirst(200, TimeUnit.MILLISECONDS)
                .subscribe(integer -> Log.d(TAG, "doThrottleFirst:" + integer));

    }
```

以上代码会发射每个 200ms 内的第一个数据。输出结果不是固定的。

比如：
2019-08-23 23:19:44.614 12605-12605/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:0
2019-08-23 23:19:44.939 12605-12605/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:3
2019-08-23 23:19:45.141 12605-12605/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:5
2019-08-23 23:19:45.361 12605-12605/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:7
2019-08-23 23:19:45.571 12605-12605/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:9

或者：
2019-08-23 23:21:27.135 12715-12715/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:0
2019-08-23 23:21:27.341 12715-12715/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:2
2019-08-23 23:21:27.565 12715-12715/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:4
2019-08-23 23:21:27.884 12715-12715/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:7

### throttleWithTimeOut

throttleWithTimeOut 通过时间来限流。每次发射一个数据就会计时。

```java
    private void doThrottleWithTimeOut() {
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                for (int i = 0; i < 10; ++i) {
                    emitter.onNext(i);
                    int sleep = 100;
                    if (i % 3 == 0) {
                        sleep = 300;
                    }
                    try {
                        Thread.sleep(sleep);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                emitter.onComplete();
            }
        })
                .throttleWithTimeout(200, TimeUnit.MILLISECONDS)
                .subscribe(integer -> Log.d(TAG, "doThrottleFirst:" + integer));
    }
```

以上代码如果是 3 的倍数就 sleep 300 毫秒，否则 sleep 100 毫秒。设定过滤时间是 200 毫秒，也就是说间隔小于 200 的会被过滤。

输出日志如下：

2019-08-23 23:32:36.480 13107-13133/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:0
2019-08-23 23:32:36.988 13107-13133/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:3
2019-08-23 23:32:37.536 13107-13133/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:6
2019-08-23 23:32:38.056 13107-13133/com.caoshen.androidsample D/RxJavaActivity: doThrottleFirst:9

## 组合操作符

组合操作符有 startWith、merge、concat、zip 和 combineLatest 等。

### startWith

startWith 操作符会在 Observable 发射的数据前面插上一些数据。

```java
    private void doStartWith() {
        Observable.just(3, 4, 5)
                .startWith(1)
                .subscribe(integer -> Log.d(TAG, "doStartWith:" + integer));
    }
```

以上代码会在最前面添加 1。

### merge

merge 操作符会将多个 Observable 合并到一个 Observable。merge 可能会让合并的 Observable 发射的数据交错。

```java
    private void doMerge() {
        Observable<Integer> observable1 = Observable.just(1, 2, 3).subscribeOn(Schedulers.io());
        Observable<Integer> observable2 = Observable.just(4, 5, 6);
        Observable.merge(observable1, observable2)
                .subscribe(integer -> Log.d(TAG, "doMerge:" + integer));
    }
```

输出结果为456123。

### concat

concat 操作符会将多个 Observable 合并到一个 Observable，并且按照顺序发射数据。

```java
    private void doConcat() {
        Observable<Integer> observable1 = Observable.just(1, 2, 3).subscribeOn(Schedulers.io());
        Observable<Integer> observable2 = Observable.just(4, 5, 6);
        Observable.concat(observable1, observable2)
                .subscribe(integer -> Log.d(TAG, "doConcat:" + integer));
    }
```

输出结果为123456。

### zip

zip 操作符合并两个或者多个 observable 发射的数据源。根据指定的函数变化它们，并且发送一个新值。

```java
    private void doZip() {
        Observable<Integer> observable1 = Observable.just(1, 2, 3);
        Observable<String> observable2 = Observable.just("a", "b", "c");
        Observable.zip(observable1, observable2, new BiFunction<Integer, String, String>() {
            @Override
            public String apply(Integer integer, String s) throws Exception {
                return integer + s;
            }
        }).subscribe(s -> Log.d(TAG, "doZip:" + s));
    }
```

以上代码将 2 个 observable 对应位置的值组合成一个字符串并返回新的 observable。

输出结果 1a、2b、3c。

### combineLatest

combineLatest 将 Observable 最新发射的数据组合在一起。

```java
    private void doCompileLatest() {
        Observable<Integer> observable1 = Observable.just(1, 2, 3);
        Observable<String> observable2 = Observable.just("a", "b", "c");
        Observable.combineLatest(observable1, observable2, new BiFunction<Integer, String, String>() {
            @Override
            public String apply(Integer integer, String s) throws Exception {
                return integer + s;
            }
        }).subscribe(s -> Log.d(TAG, "doCompileLatest:" + s));
    }
```

以上代码 observable1 发射数据时，observable2 还没有发射数据，combineLatest 将 observable1 的最新数据 3 和 observable2 的每个数据组合。

输出结果：3a、3b、3c。

## 辅助操作符

辅助操作符可以帮助我们更加方便地处理 Observable。辅助操作符包括 delay、do、subscribeOn、observeOn、timeout、materialize、timeInterval、timestamp 和 to 等。

这里介绍 delay、do、subscribeOn、observeOn、timeout。

### delay

delay 可以在 Observable 发射每项数据之前都暂停一段指定的时间段，即延迟发射。

```java
    private void doDelay() {
        Observable.create(new ObservableOnSubscribe<Long>() {
            @Override
            public void subscribe(ObservableEmitter<Long> emitter) throws Exception {
                emitter.onNext(System.currentTimeMillis() / 1000);
            }
        })
                .delay(2, TimeUnit.SECONDS)
                .subscribe(aLong -> Log.d(TAG, "delay:" + (System.currentTimeMillis() / 1000 - aLong)));
    }
```

以上代码输出 2，延迟 2 秒发射。

### do

do 系列操作符为原始 Observable 的生命周期事件注册一个回调。当 Observable 的某个事件发生时就会调用这些回调。RxJava 中有很多 Do 系列操作符，如下所示：

1. doOnEach 当 Observable 每发射一项数据时就会调用它一次，包括 onNext、onCompleted、onError。
2. doOnNext 当 onNext 时会被调用。
3. doOnSubscribe 当观察者订阅 observable 时会被调用
4. doOnUnsubscribe 当观察者取消订阅时会被调用，observable 通过 onError 或者 onCompleted 结束时，会取消订阅所有的 subscriber。
5. doOnCompleted 当 observable 正常终止调用 onCompleted 时会被调用。
6. doOnError 当 observable 异常终止调用 onError 时会被调用。
7. doOnTerminate 当 observable 终止，无论是正常终止还是异常终止之前都会被调用。
8. finallyDo 当 observable 终止，无论是正常终止还是异常终止之后都会被调用。

```java
    private void doOnNext() {
        Observable.just(1, 2)
                .doOnNext(integer -> Log.d(TAG, "doOnNext:" + integer))
                .subscribe(integer -> Log.d(TAG, "onNext:" + integer),
                        throwable -> Log.d(TAG, "onError:" + throwable.getMessage()),
                        () -> Log.d(TAG, "onCompleted"));
    }
```

以上代码输出：

```
doOnNext:1
onNext:1
doOnNext:2
onNext:2
onCompleted
```

### subscribeOn 和 observeOn

subscribeOn 用于指定 observable 自身在哪个线程上运行。

observeOn 用于指定 observable 所运行的线程，也就是发射出的数据在哪个线程上使用。一般情况下会指定在主线程中运行，这样就可以修改 UI。

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

### timeout

如果原始 Observable 过了指定的一段时长没有发射任何数据，timeout 操作符会以一个 onError 通知终止这个 Observable，或者继续执行一个备用的 Observable。timeout 有很多种变体。这里介绍其中的一种：timeout(long, TimeUnit, Observable)，它会在超时的时候切换到使用另一个备用的 Observable，而不是发送错误通知。

```java
    private void doTimeout() {
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                for (int i = 0; i < 4; ++i) {
                    try {
                        Thread.sleep(i * 100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    emitter.onNext(i);
                }
                emitter.onComplete();
            }
        }).timeout(200, TimeUnit.MILLISECONDS, Observable.just(10, 11))
                .subscribe(integer -> Log.d(TAG, "doTimeout:" + integer));
    }
```

以上代码设置了 200 ms 的超时，超时后发送 10 和 11。

输出如下：

```
doTimeout:0
doTimeout:1
doTimeout:10
doTimeout:11
```

也有可能是：

```
doTimeout:0
doTimeout:1
doTimeout:2
doTimeout:10
doTimeout:11
```

## 错误处理操作符

RxJava 在错误出现的时候就会调用 Subscriber 的 onError 方法将错误分发出去，有 Subscriber 自己来处理错误。如果每个 Subscriber 都处理一遍的话，工作量大，这时候可以使用错误操作符。错误操作符有 catch 和 retry。

### catch

catch 操作符拦截原始 Observable 的 onError 通知，将它替换成其他数据项或数据序列，让产生的 Observable 能够正常终止或者根本不终止。RxJava 将 catch 实现为以下 3 个不同的操作符。

1. onErrorReturn
2. onErrorResumeNext
3. onExceptionResumeNext

onErrorReturn: Observable 遇到错误时返回备用的 Observable，备用 Observable 会忽略原有 Observable 的 onError 调用，不会将错误传递给观察者。作为替代，它会发射一个特殊的项并且调用观察者的 onCompleted 方法。

onErrorResumeNext: Observable 遇到错误时返回备用的 Observable，备用 Observable 会忽略原有 Observable 的 onError 调用，不会将错误传递给观察者。作为替代，它会发射备用的 Observable 数据。

onExceptionResumeNext: 它和 onErrorResumeNext 类似。不同的是，如果 onError 收到的 Throwable 不是一个 Exception，它会将错误传递给观察者的 onError 方法，不会使用备用的 Observable。

这里用 onErrorReturn 操作符来举例。

```java
    private void doOnErrorReturn() {
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                for (int i = 0; i < 5; ++i) {
                    if (i > 3) {
                        emitter.onError(new Throwable("throwable"));
                    }
                    emitter.onNext(i);
                }
                emitter.onComplete();
            }
        }).onErrorReturn(throwable -> 6)
                .subscribe(integer -> Log.d(TAG, "onNext:" + integer),
                        throwable -> Log.d(TAG, "onError:" + throwable.getMessage()),
                        () -> Log.d(TAG, "onComplete:"));
    }
```

以上代码当 i = 4 时抛出异常，onErrorReturn 返回 6。

输出如下：

```
onNext:0
onNext:1
onNext:2
onNext:3
onNext:6
onComplete:
```

注意如果以上代码改为 i > 2，那么当 i = 3 和 4 时都会抛出异常，这是程序会 crash。抛出异常如下：
io.reactivex.exceptions.UndeliverableException: The exception could not be delivered to the consumer because it has already canceled/disposed the flow or the exception has nowhere to go to begin with.

### retry

retry 操作符不会将原始的 Observable 的 onError 通知传递给观察者，它会订阅这个 Observable，再给它一次机会无错误地完成其数据序列。

retry 总是传递 onNext 给观察者，由于重新订阅，可能造成数据项重复。

Rxjava 中的实现为 retry 和 retryWhen。

这里以 retry(long) 来举例，它指定最多重新订阅的次数。如果次数超了，它不会重新订阅，而会把最新的一个 onError 通知传递给自己的观察者。

```java
    private void doRetry() {
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                for (int i = 0; i < 5; ++i) {
                    if (i == 1) {
                        emitter.onError(new Throwable("throwable"));
                    } else {
                        emitter.onNext(i);
                    }
                }
                emitter.onComplete();
            }
        }).retry(2)
                .subscribe(integer -> Log.d(TAG, "onNext:" + integer),
                        throwable -> Log.d(TAG, "onError:" + throwable.getMessage()),
                        () -> Log.d(TAG, "onComplete:"));
    }
```
以上代码会重试 2 次。

输出如下：

```
onNext:0
onNext:0
onError:throwable
```
多次重复依然如此。少走了一次 onNext: 0。

如果以上代码改为 if (i == 2)，输出如下：

```
onNext:0
onNext:1
onNext:0
onNext:1
onNext:0
onNext:1
onError:throwable
```

## 条件操作符和布尔操作符

条件操作符和布尔操作符可用于根据条件发射或变换 Observable，或者对它们做布尔运算。

### 条件操作符

条件操作符有 amb、defaultIfEmpty、skipUntil、skipWhile、takeUntil 和takeWhile 等。这里介绍前两种：

#### amb (ambArray)

amb 操作符对于给定的两个或者多个 Observable，它只发射首先发射数据或通知的那个 Observable 的所有数据。

```java
    private void doAmpArray() {
        Observable.ambArray(Observable.just(1, 2, 3).delay(2, TimeUnit.SECONDS),
                Observable.just(4, 5,6))
                .subscribe(integer -> Log.d(TAG, "ambArray:" + integer));
    }
```
输出如下：

```
ambArray:4
ambArray:5
ambArray:6
```

#### defaultIfEmpty

发射来自原始 Observable 的数据。如果原始 Observable 没有发射数据，就发射一个默认数据，如下所示：

``` java
    private void doDefaultIfEmpty() {
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                emitter.onComplete();
            }
        }).defaultIfEmpty(3)
                .subscribe(integer -> Log.d(TAG, "defaultIfEmpty:" + integer));
    }
```

以上代码没有发射数据就返回 3。

输出如下：

```
defaultIfEmpty:3
```

### 布尔操作符

布尔操作符有 all、contains、isEmpty、exists 和 sequenceEqual，这里介绍前 3 个操作符。

#### all

all 操作符用一个函数对所有数据做判断，如果每个都满足，则返回 true，否则返回 false。

```java
    private void doAll() {
        Observable.just(1, 2, 3, 4)
                .all(integer -> integer < 3)
                .subscribe(aBoolean -> Log.d(TAG, "all:" + aBoolean));
    }
```

以上代码返回是否所有的数都小于 3。

输出如下：

```
all:false
```

#### contains 和 isEmpty

contains 操作符用来判断源 Observable 所发射的数据是否包含某一个数据。如果包含该数据，会返回 true；如果 Observable 已经结束了却还没有发射这个数据，则返回 false。

isEmpty 与 contains 完全相反。

```java
    private void doContains() {
        Observable.just(1, 2, 3).contains(1).subscribe(aBoolean -> Log.d(TAG, "contains:" + aBoolean));
        Observable.just(1, 2, 3).isEmpty().subscribe(aBoolean -> Log.d(TAG, "isEmpty:" + aBoolean));
    }
```

输出如下：

```
contains:true
isEmpty:false
```

## 转换操作符

转换操作符用来将 Observable 转换成为另一个对象或者数据结构，转换操作符有 toList、toSortedList、toMap、toMultiMap、getIterator 和 nest 等，这里介绍前 3 种。

### toList

toList 将多个数据转换成一个 list。

```java
    private void doToList() {
        Observable.just(1, 2, 3).toList().subscribe(integers -> {
            for (int i : integers) {
                Log.d(TAG, "toList:" + i);
            }
        });
    }
```

输出如下：

```
toList:1
toList:2
toList:3
```

### toSortedList

toSortedList 类似 toList，但是它是有序的。

```java
    private void doToSortedList() {
        Observable.just(3, 1, 2).toSortedList().subscribe(integers -> {
            for (int i : integers) {
                Log.d(TAG, "toSortedList:" + i);
            }
        });
    }
```

输出如下：

```
toSortedList:1
toSortedList:2
toSortedList:3
```

### toMap

toMap 操作符将发射的所有数据集合到一个 Map （默认 HashMap），然后发射这个 Map。

```java
    private void doToMap() {
        Observable.just("apple", "orange", "banana", "oreo", "pie", "pear")
                .toMap(s -> s.charAt(0))
                .subscribe(m -> Log.d(TAG, "toMap:" + m.toString()));
    }
```

输出如下：

```
toMap:{p=pear, a=apple, b=banana, o=oreo}
```

以上代码可以看出 o 开头的是排在后面的 oreo，p 开头的是排在后面的 pear。