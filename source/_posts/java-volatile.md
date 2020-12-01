---
title: Java volatile 关键字
date: 2019-10-04 19:31:29
tags: 多线程
categories: Java
---

# Java volatile 关键字

volatile 是 Java 的关键字，它的本意是易变的，即被 volatile 声明的变量可能被其他线程修改，需要用 volatile 保证变量的可见性和有序性。

有时仅仅为了读写一个或者两个实例域就使用同步的话，显得开销过大；而 volatile 关键字为实例域的同步访问提供了免锁机制。如果声明一个域为 volatile，那么编译器和虚拟机就知道该域是可能被另一个线程并发更新的。

## Java 内存模型

Java 中的堆内存用来存储对象实例，堆内存是被所有线程共享的运行时内存区域，因此它存在内存可见性的问题。而局部变量、方法定义的参数则不会在线程之间共享，它们不会有内存可见性问题，也不受内存模型的影响。Java 内存模型定义了线程和主存之间的抽象关系：线程之间的共享变量存储在主存中，每个线程有一个私有的本地内存，本地内存中存储了该线程共享变量的副本。

需要注意的是本地内存是 Java 内存模型的一个抽象概念，其并不真实存在，它涵盖了缓存、写缓冲区、寄存器等区域。Java 内存模型控制线程之间的通信，它决定一个线程对主存共享变量的写入何时对另一个线程可见。

线程 A 和线程 B 之间若要通信的话，必须要经历下面两个步骤：

1. 线程 A 把线程 A 本地内存中更新过的共享变量刷新到主存中去；
2. 线程 B 到主存中去读取线程 A 之前已更新过的共享变量。

## 原子性、可见性和有序性

原子性、可见性和有序性是并发编程中的三个特性。

### 原子性

对基本类型的读取和赋值操作是原子性的操作，即这些操作是不可中断的，要么执行完毕，要么不执行。

```java
x = 3;
y = x;
x++;
```

上面 3 个语句只有第 1 条语句是原子性操作，其他 2 个语句都不是原子性操作。

第 2 条语句虽然是赋值操作，但是它包含了 2 个步骤，首先读取 x 的值，然后将 x 的值写入工作内存。读取 x 的值以及将 x 的值写入工作内存这两个操作单拿出来都是原子操作，但是合起来就不是原子操作。

第 3 条语句包括了 3 个操作：读取 x 的值、对 x 的值进行加 1、向工作内存写入新值。

通过这 3 个语句我们得知，一个语句包含多个操作时，就不是原子性操作，只有简单地读取和赋值（将数字赋值给某个变量）才是原子性操作。

java.util.concurrent.atomic 包中有很多类使用了高效的机器级指令（而不是使用锁）来保证操作的原子性。例如 AtomicInteger 类提供了方法 incrementAndGet 和 decrementAndGet，它们分别以原子方式将一个整数自增和自减。可以安全地使用 AtomicInteger 类作为共享计数器而无需同步。另外这个包还包含了 AtomicBoolean、AtomicLong、AtomicReference 这些原子类。

### 可见性

可见性是指线程之间地可见性，一个线程修改地状态对另一个线程时可见地。也就是说一个线程修改地结果，另一个线程马上就能看到。当一个共享变量被 volatile 修饰时，它会保证修改地值立即被更新到主存，同时其他线程读取共享变量时，会直接从主存中读取，而不是从线程工作内存读取，所以对其他线程是可见的。而普通的共享变量不能保证可见性，因为普通共享变量被修改之后，并不会立即被写入主存，何时被写入主存也是不确定的。当其他线程去读取该值时，此时主存中可能还是原来的旧值，这样就无法保证可见性。

### 有序性

Java 内存模型中允许编译器和处理器对指令进行重排序，虽然重排序过程不会影响到单线程执行的正确性，但是会影响到多线程并发执行的正确性。这时可以通过 volatile 来保证有序性。除了 volatile，也可以通过 synchronized 和 lock 来保证有序性。我们知道，synchronized 和 lock 保证每个时刻只有一个线程执行同步代码，这相当于是让线程顺序执行同步代码，从而保证了有序性。

## volatile 关键字

当一个共享变量被 volatile 修饰之后，就具备了两个含义，一个是线程修改了变量的值时，变量的新值对其他线程时立即可见的。另一个含义是禁止指令重排序。重排序通常是编译器或者运行时环境为了优化程序性能而采取的对指令进行重新排序执行的一种手段。重排序分为两类：编译期重排序和运行期重排序，分别对于编译时和运行时环境。

### volatile 不保证原子性

我们知道 volatile 保证了操作的可见性，但是对于一些自增操作，它无法保证原子性。

```java
public class VolatileTest {
    public volatile int inc = 0;

    public void increase() {
        inc++;
    }

    public static void main(String[] args) {
        final VolatileTest volatileTest = new VolatileTest();
        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int j = 0; j < 1000; ++j) {
                        volatileTest.increase();
                    }
                }
            }).start();
        }
        // 如果有子线程就让出资源，保证所有子线程都执行完
        while (Thread.activeCount() > 2) {
            Thread.yield();
        }
        System.out.println(volatileTest.inc);
    }
}
```

以上代码输出如下：

```
9475
```

可以看出并非每次都正确执行了自增操作。自增操作是不具备原子性的，它包括 3 个步骤，读取变量的原始值、进行加 1、写入工作内存。也就是说，自增操作的 3 个自操作可能分割开执行。

### volatile 保证有序性

volatile 关键字能禁止指令重排序，因此 volatile 能保证有序性。volatile 关键字禁止指令重排序有 2 个含义：一个是当程序执行到 volatile 变量的操作时，在其前面的操作已经全部执行完毕，并且结果会对后面的操作可见，在其后面的操作还没有运行；在进行指令优化时，在 volatile 变量之前的语句不能在 volatile 变量后面执行；同样，在 volatile 变量之后的语句也不能再 volatile 变量之前执行。

## 正确使用 volatile 关键字

通常来说，使用 volatile 必须具备以下两个条件：

1. 对变量的写操作不会依赖于当前值。
2. 该变量没有包含在具有其他变量的不变式中。

第一个条件就是不能是自增、自减操作，上文已经提到 volatile 不保证原子性。

关于第二个条件，我们来举一个例子，它包含了一个不等式：下界总是小于或者等于上界。

```java
public class NumberRange {
    private volatile int lower, upper;

    public int getLower() {
        return lower;
    }

    public int getUpper() {
        return upper;
    }

    public void setLower(int value) throws IllegalAccessException {
        if (value > upper) {
            throw new IllegalAccessException("great than upper");
        }
        lower = value;
    }

    public void setUpper(int value) throws IllegalAccessException {
        if (value < lower) {
            throw new IllegalAccessException("less than lower");
        }
        upper = value;
    }

    public static void main(String[] args) throws IllegalAccessException {
        NumberRange numberRange = new NumberRange();
        numberRange.setLower(0);
        numberRange.setUpper(5);
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    numberRange.setLower(4);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    numberRange.setUpper(3);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```

以上代码 range 的初始值是 0 到 5，假如在同一时刻，线程 1 调用 setLower(4)，线程 2 调用 setUpper(3)，那么新的范围是 (4, 3)。这显然是不对的，因此无法用 volatile 实现 setLower 和 setUpper 操作的原子性。

使用 volatile 有很多种场景，这里介绍其中 2 种：

### 状态标志

```java
    volatile boolean shutdownRequested;
    
    public void shutdown() {
        shutdownRequested = true;
    }
    
    public void doWork() {
        while (!shutdownRequested) {
            // do work
        }
    }
```

volatile boolean 标记保证了变量的可见性，对于变量的修改对每个线程都是可见的。

### 双重校验锁（DCL）

```java
public class Singleton {
    private static volatile Singleton singleton = null;

    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

因为 singleton = new Singleton(); 这个操作会先分配内存，然后调用构造方法构造 Singleton，最后调用赋值操作。这三个操作可能被指令重排序，导致不按顺序执行，线程 1 还没将构造的 singleton 写入内存，线程 2 就去内存读取，读取出错误的值。加上 volatile 关键字之后会产生一个内存屏障，保证操作按照顺序执行。

## 总结

与锁相比，volatile 变量是一种非常简单但同时又非常脆弱的同步机制，它在某些情况想将提供优于锁的性能和伸缩性。如果严格遵守 volatile 的使用条件，即变量真正独立于其他变量和自己以前的值，在某些情况下可以使用 volatile 代替 synchronized 来简化代码。但是，使用 volatile 的代码比使用锁的代码更加容易出错。使用状态标志和双重校验锁是 volatile 的两种常见用例，其他情况下最好还是使用 synchronized 关键字。