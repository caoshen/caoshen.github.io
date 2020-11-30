---
title: Android 源码的享元模式
date: 2019-11-25 23:53:57
tags: 设计模式
categories: Android
---

# Android 源码的享元模式

## 享元模式介绍

享元模式是对象池的一种实现，它的英文名称叫做 Flyweight，代表轻量级的意思。享元模式用来尽可能减少内存使用量，它适合用于可能存在大量重复对象的场景，来缓存可共享的对象，达到对象共享、避免创建过多对象的效果，这样一来就可以提升性能、避免内存移除等。

## 享元模式定义

使用共享对象可有效地支持大量地细粒度的对象。

## Android 源码中的享元模式

Handler 消息机制中的 Message 消息池就是使用享元模式复用了 Message 对象。

使用 Message 时一般会用到 Message.obtain 来获取消息。如果使用 new Message() 会构造大量的 Message 对象。

obtain 方法如下：

```java
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                // 清空 in-use 标记
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

sPoolSync 和 sPool 定义如下：

```java
    // sometimes we store linked lists of these things
    /*package*/ Message next;


    /** @hide */
    public static final Object sPoolSync = new Object();
    private static Message sPool;
```

sPoolSync 是一个对象锁，用于在获取 Message 对象时进行同步锁。

sPool 是一个静态的 Message 对象。

next 是一个 Message 对象，指向下一个 Message。

可以看出，Message 消息池没有使用 map 这样的容器，而是使用的链表。

那么这些 Message 是什么时候放入链表中的呢？我们在 obtain 函数中只看到了从链表中获取，并且看到存储。如果消息池链表中没有可用对象的时候，obtain 中则是直接返回一个通过 new 创建的 Message 对象，而且并没有存储到链表中。

Message 类有一个 recycle 方法，它用来回收消息，并且把回收掉的消息添加到对象池链表中。

recycle 方法如下：

```java
    public void recycle() {
        // 判断消息是否还在使用
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        // 清空状态，并且将消息添加到消息池中
        recycleUnchecked();
    }
```

recycleUnchecked 方法如下：

```java
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        // 清空消息状态，设置该消息 in-use flag
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        // 回收消息到消息池中
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

recycle 会将一个 Message 回收到一个全局的池。如果消息在使用就抛出异常，否则调用 recycleUnchecked。

recycleUnchecked 先清空字段，然后回收消息，将 sPool 指向当前消息，同时 size 加一。

Message 通过在内部构建一个链表来维护一个被回收的 Message 对象的对象池，当用户调用 obtain 时会优先从池中取，如果池中没有可以复用的对象则创建这个新的 Message 对象。这些新创建的 Message 对象在被使用完之后会被回收到这个对象池中，当下次再调用 obtain 时，它们就会被复用。

因为 Android 应用是事件驱动的，因此，如果通过 new 创建 Message 会产生大量的重复的 Message 对象，导致内存占用率高、频繁 GC 等问题，通过享元模式创建一个大小为 50 的消息池，避免了上述问题。

## 参考资料

《Android 源码设计模式解析与实战 · 第22章 对象共享，避免创建多对象——享元模式》
