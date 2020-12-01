---
title: Java 同步方法和同步代码块
date: 2019-10-03 16:15:53
tags: 多线程
categories: Java
---

# Java 同步方法和同步代码块

Lock 和 Condition 接口提供了高度定制的锁定控制，但是大多数情况下，并不需要那样的控制，并且可以使用一种嵌入到 Java 语言内部的机制。

从 Java 1.0 开始，Java 中的每个对象都有一个内部锁。如果一个方法用 synchronized 关键字声明，那么对象的锁将保护整个方法。也就是说，要调用该方法，线程必须获得内部的对象锁。

## 同步方法

使用 synchronized 关键字等价于用 ReentrantLock 控制 lock 和 unlock。

```java
    public synchronized void dosome() {
        // do some
    }
```

等价于：

```java
    ReentrantLock lock = new ReentrantLock();
    
    public void dosome() {
        lock.lock();
        try {
            // do some
        } finally {
            lock.unlock();
        }
    }
```

内部对象锁只有一个相关条件，wait 方法将一个线程添加到等待集，notifyAll 或者 notify 方法解除等待线程的阻塞状态。也就是说 wait 相当于调用 condition.await()，notifyAll 相当于调用 condition.signalAll()。

```java
    public synchronized void transferLocked(int from, int to, int amount) throws InterruptedException {
        while (accounts[from] < amount) {
            wait();
        }
        accounts[from] -= amount;
        accounts[to] += amount;
        notifyAll();
    }
```

可以看出使用 synchronized 关键字编写代码要简洁的多。要理解这一代码，需要了解每个对象都有一个内部锁，并且该锁有一个内部条件。有该锁来管理那些试图进入 synchronized 方法的线程，由该锁的条件来管理那些调用 wait 的线程。

## 同步代码块

除了使用同步方法来获得锁，还可以使用同步代码块来获得锁。

```java
    public void dosomeLocked() {
        synchronized (obj) {
            // do some
        }
    }
```

方法获得了 obj 锁，obj 是一个对象。

```java
    private Object obj = new Object();

    public void doTransferLocked(int from, int to, int amount) {
        synchronized (obj) {
            accounts[from] -= amount;
            accounts[to] += amount;
        }
    }
```

这里创建了一个名为 Lock 的 Object 类，为的是使用 Object 类所持有的锁。

同步代码块锁住的范围比同步方法要小，如果同步方法更适合你的程序，尽量使用同步方法，减少出错的概率。如果特别需要使用 Lock 和 Condition 结构提供的独有特性，才使用 Lock 和 Condition。Lock 和 Condition 相比 synchronized 关键字更灵活。

