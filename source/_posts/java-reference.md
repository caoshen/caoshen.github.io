---
title: Java 强软弱虚引用
date: 2019-10-04 12:23:59
tags: Java
categories: Java
---

# Java 强软弱虚引用

Java 根据对象的引用方式可以分为：强引用、软引用、弱引用、虚引用。
即 SoftReference、WeakReference、PhantomReference。
SoftReference、WeakReference、PhantomReference 位于 java.lang.ref 包，它们通常和 ReferenceQueue 一起使用。

## 强引用

强引用即对象的直接引用。比如：

Handler handler = new Handler();

强引用的对象不会被系统回收。如果内存不足也不会回收强引用对象，JVM 可能抛出 OOM 错误，程序异常终止。
强引用特点是可以直接访问对象，可能导致内存泄露。

## 软引用 SoftReference

如果一个对象具有软引用，那么当内存足够时，系统不会回收对象；如果内存不够，系统会回收这些对象。

软引用可以和 ReferenceQueue 队列关联。

        ReferenceQueue<String> queue = new ReferenceQueue<>();
        SoftReference<String> softReference = new SoftReference<>("abc", queue);

如果 SoftReference get() 的对象为空，说明它已被系统回收。

## 弱引用 WeakReference

如果一个对象具有弱引用，那么不管内存是否够用，只要触发垃圾回收，就会回收弱引用的对象。
Android 里面弱引用 WeakReference 通常和 Handler 连用，避免 Activity 销毁之后，工作线程还在执行，并且使用持有 Activity 强引用的 Handler 发送回调消息。
从而导致  Activity 声明周期销毁了但是 Activity 对象没有被回收。

因此改为 Handler 持有 Activity 的弱引用。

```java
private static class TimeHandler extends Handler {
        private final WeakReference<TimeOutActivity> mReference;

        TimeHandler(TimeOutActivity activity) {
            mReference = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            TimeOutActivity activity = mReference.get();
            if (activity == null) {
                return;
            }
            if (msg.what == MSG_CHECK_TIME) {
                activity.checkTime();
            }
        }
    }
```

对比软引用和弱引用，可以发现弱引用的对象具有更短的生命周期，更容易被系统回收。

## 虚引用 PhantomReference

虚引用不会影响对象的声明周期。它用来观察所引用对象是否被回收。
虚引用的 get 方法总是返回 null，而与它的引用对象无关。
构造 PhantomReference 时必须传入 ReferenceQueue。

```java
public class PhantomReference<T> extends Reference<T> {
...
    public T get() {
        return null;
    }
...
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }

}
```
