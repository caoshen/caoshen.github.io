---
title: 用 RxJava 实现 RxBus
date: 2019-09-01 22:15:05
tags: RxJava
categories: Android
---

# 用 RxJava 实现 RxBus

RxJava 可以用来实现 RxBus，实现事件发送和监听。

## 创建 RxBus

```java
public class RxBus {

    private static volatile RxBus rxBus;

    private final Subject<Object> subject = PublishSubject.create().toSerialized();

    private RxBus() {

    }

    public static RxBus getInstance() {
        if (rxBus == null) {
            synchronized (RxBus.class) {
                if (rxBus == null) {
                    rxBus = new RxBus();
                }
            }
        }
        return rxBus;
    }

    public void post(Object object) {
        subject.onNext(object);
    }

    public <T> Observable<T> toObservable(Class<T> eventType) {
        return subject.ofType(eventType);
    }
}
```

这里使用双重校验锁（DCL）加上 volatile 关键字的方式实现 RxBus 单例。

使用 PublishSubject.create().toSerialized() 方法得到一个线程安全的 Subject。

toSerialized 方法如下：

```java
    @NonNull
    public final Subject<T> toSerialized() {
        if (this instanceof SerializedSubject) {
            return this;
        }
        return new SerializedSubject<T>(this);
    }
```

在 RxJava2 中 SerializedSubject 类不再是 public 修饰的，没有对外暴露，而是使用 toSerialized 创建。

post 方法用来发送事件，它直接调用了 onNext 方法。

```java
    public void post(Object object) {
        subject.onNext(object);
    }
```

toObservable 用来把事件类型转换为 Observable 类型。

```java
    public <T> Observable<T> toObservable(Class<T> eventType) {
        return subject.ofType(eventType);
    }
```

ofType 方法代码如下：

```
    public final <U> Observable<U> ofType(final Class<U> clazz) {
        ObjectHelper.requireNonNull(clazz, "clazz is null");
        return filter(Functions.isInstanceOf(clazz)).cast(clazz);
    }
```

可以看出先通过 instanceOf 判断是否是指定类型，然后用 filter 过滤指定类型的事件，最后使用 cast 将 Observable 转换为指定类型的 Observable。

## 发送事件

```java
public class RxBusActivity extends FragmentActivity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_rx_bus);
        Button btn = findViewById(R.id.rx_btn);
        btn.setOnClickListener(v -> {
            RxBus.getInstance().post(new MessageEvent("用 RxJava 实现 RxBus"));
        });
    }
}
```

点击按钮时，用调用 post 方法发送事件。

## 接收事件

```java
public class RxBusFragment extends Fragment {

    private TextView text;
    private Disposable disposable;

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_rx_bus, container, false);
        text = view.findViewById(R.id.frag_text);
        return view;
    }

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        disposable = RxBus.getInstance().toObservable(MessageEvent.class).subscribe(messageEvent -> {
            if (messageEvent != null) {
                text.setText(messageEvent.getMessage());
            }
        });
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        if (disposable != null && !disposable.isDisposed()) {
            disposable.dispose();
        }
    }
}
```

在 Fragment 的 onActivityCreated 方法中订阅事件，如果收到事件就改变 textView 文字。

## 取消订阅

为了避免内存泄漏，需要在订阅者的退出回调 onDestroy 中取消事件。

```java
    @Override
    public void onDestroy() {
        super.onDestroy();
        if (disposable != null && !disposable.isDisposed()) {
            disposable.dispose();
        }
    }
```

## 总结

为了实现事件的发送和订阅，可以使用 RxJava 的 PublishSubject.create().toSerialized() 方法得到一个线程安全的 Subject，然后分别在合适的地方订阅（subject subscribe）和发送（subject onNext）事件。为了避免内存泄漏，需要在合适的地方取消订阅（dispose）。