---
title: Android 消息机制
date: 2019-08-19 23:10:41
tags: Handler
categories: Android
---

# Android 消息机制

Android 的消息机制是通过 Handler 来完成的。Handler 内部有 Looper 和 MessageQueue 分别用来处理消息循环和保存消息队列。

线程默认是没有 Looper 的，因此没有办法处理消息。线程要使用 Handler 处理消息，必须先准备消息循环，即先创建 Looper，再根据 Looper 创建线程的 Handler。

应用主线程(UI 线程) ActivityThead 在启动的时候已经在 main 方法中创建了 MainLooper，所以可以直接在主线程中创建 Handler 而不会抛出异常。

## 消息机制概述

Android 消息机制主要依靠 Handler 来发送和处理消息（Message）。Handler 的主要作用是将一个任务切换到某个指定的线程，比如在子线程执行完耗时任务后，通过 Handler 把任务执行的结果传递到主线程中，更新界面。Handler 也用来处理顺序消息或者处理延时消息。

Android 规定 UI 只能在主线程中进行，大多数时候，如果直接在子线程中访问 UI，那么程序会抛出异常。这个异常是由 ViewRootImpl 的 checkThread 方法完成验证然后抛出的（如果在 checkThread 方法之前直接修改了 UI，是不会抛出异常的，这个时候还没有触发 UI 验证）。

如果主线程有耗时操作阻碍了 UI 线程刷新，应用就会无法响应用户操作，出现 ANR (Application Not Response)。因此耗时操作会放在子线程执行。但是如果直接允许了子线程和主线程同时可以操作 UI，会导致 UI 出现不可控制的现象，比如两个线程同时修改界面上的文字，界面文字就会混乱。为什么系统不对 UI 控件的访问加上锁机制？缺点有两个：1. 加锁机制会让 UI 访问逻辑变得复杂；2. 加锁机制会降低 UI 访问的效率，因为锁机制会阻塞某些线程的执行。

## Handler 创建

Handler 的构造函数如下：

```java
    public Handler() {
        this(null, false);
    }

    public Handler(Callback callback) {
        this(callback, false);
    }

    public Handler(Looper looper) {
        this(looper, null, false);
    }

    public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);
    }

    public Handler(boolean async) {
        this(null, async);
    }

    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

可以看出由 looper 、callback、async 三个部分组成。looper 用来处理消息循环，callback 是处理完消息的回调，async 表示是否为异步消息。

在构造 Handler 的时候，looper、callback、async 都可以不传递参数。如果不传 looper 值，Handler 会使用 Looper.myLooper() 方法查找所在线程里面是否存在创建好的 looper，如果没有 looper，直接抛出 RuntimeException，告知当前线程没有调用 Looper.prepare() 方法。如果不传 callback 值，默认处理完消息后没有回调。如果不传 async，async 默认为 false，表示 Handler 里面发送的都是同步消息。

## ThreadLocal 的工作原理

ThreadLocal 线程局部变量是一个线程内部的数据存储类，通过它可以在指定的线程存储数据，存储数据之后，只能在指定的线程获取数据，其他的线程无法获取。

ThreadLocal set 

```java
    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

set 方法首先通过 currentThread() 得到当前线程，然后从得到当前线程的 ThreadLocalMap。如果 map 不为空，就把值 set 到 ThreadLocalMap；如果 map 为空，就新建一个 ThreadLocalMap，在 ThreadLocalMap 的构造方法里面有一个 Entry 类型的 table 数组，然后将 threadLocal 和 value 组成一个 Entry 放到 table 数组中。

ThreadLocal get

```java
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

get 方法先用 currentThread() 得到当前线程，然后得到当前线程的 ThreadLocalMap。如果当前线程的 ThreadLocalMap 里面有 ThreadLocal 对应的 Entry 值，而且不为 null，就强制转换为泛型类的类型。如果 ThreadLocalMap 查找到的 Entry 为 null，就返回 setInitialValue() 的初始值。

```java
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

setInitialValue() 方法会先获取 initialValue() 的返回值，然后和 ThreadLocal 的 set 方法类似，将 initialValue 值 set 到 ThreadLocalMap 的 table 中。

```java
    protected T initialValue() {
        return null;
    }
```

initialValue() 方法返回 null。但是它是 protected 的，可以被继承。

从 ThreadLocal 的 get 和 set 方法可以看出，它们首先都调用了 Thread.currentThread() 方法获取当前线程，然后得到当前线程的 ThreadLocalMap，所操作的对象都是线程的 ThreadLocalMap 里面的 table 数组，因此对于 ThreadLocal 的读写都仅限于各自线程的内部。所以 ThreadLocal 可以在多个线程中互不干扰的读写数据。

## 消息队列（MessageQueue）的工作原理

Handler 的 sendMessage 和 post 方法都是将一个 Message 放入 MessageQueue，在消息循环的过程中从 MessageQueue 取出消息并分发消息。

```java
    public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
    }

    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }

    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

可以看出最终会调用 enqueueMessage 方法，把 Message 的 target 设置为当前的 Handler，target 就是处理这条 Message 的 Handler，所以 Handler 的 handleMessage 方法会把 Message 作为参数，使用的时候只需要复写 handleMessage 即可。

```java
if (mAsynchronous) {
    msg.setAsynchronous(true);
}
```
Handler 构造方法中的 asynchronous 如果设置为 true，那么它所有发送的消息都是异步消息，即 message 的 flags 属性会多一个 FLAG_ASYNCHRONOUS。

最后，消息和它触发的事件一起传递给 MessageQueue 的 enqueueMessage 方法。

```java
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

enqueueMessage 的原理是通过一个无限循环，找到入队的 Message 该插入的位置，这个位置是由 Message 的 when 属性决定的，即它的触发时间。

next() 方法取出一条消息并将它从消息队列移除。

```java
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                ...
            }

            ...
        }
    }
```

可以看出 next() 方法也是开启一个无限循环，如果当前事件还没有到当前消息的触发时间，消息队列会一直阻塞直到触发时间到来；如果已经到了触发时间，那么会将消息从消息队列移除（链接到下一个消息），然后返回这个消息。

## SyncBarrier 和异步消息

enqueueMessage 的过程中有一个 SyncBarrier（同步屏障） 和唤醒消息队列的场景。

```java
needWake = mBlocked && p.target == null && msg.isAsynchronous();
```
如果队列被阻塞（mBlocked = true），队列当前处理的消息的 target 为 null，而且入队的消息是一个异步消息时，说明队列需要从被阻塞的状态下唤醒。

之前查看 Handler 的 enqueueMessage 方法时，可以发现每个 Message 都有 target，而且这个 target 就是发送消息的 Handler。但是有一种情况下，Message 是没有 target 的。这种情况就是 postSyncBarrier。

```java
private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```

可以看出 postSyncBarrier 的消息相比 enqueueMessage 的消息少了一个 target。在 MessageQueue 的 next 方法中，就是通过 message 是否有 target 来判断消息队列里是否存在 SyncBarrier。

```java
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
```

如果有 SyncBarrier，那么后续的同步消息都不会被处理，直到遇到一个异步消息。

