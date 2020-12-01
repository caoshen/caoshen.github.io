---
title: Java 线程的中断与停止
date: 2019-10-01 15:27:00
tags: 多线程
categories: Java
---

# Java 线程的中断与停止

当线程的 run 方法执行完毕，或者在方法中出现了没有捕获的异常时，线程将终止。

## 被弃用的 stop 方法

Thread 有一个 stop 方法，但是 stop 方法被废弃了。Thread 里面的 suspend\resume 也被废弃了。JDK 是这么解释的：

```java
    /**
     ...
     * @deprecated This method is inherently unsafe.  Stopping a thread with
     *       Thread.stop causes it to unlock all of the monitors that it
     *       has locked (as a natural consequence of the unchecked
     *       {@code ThreadDeath} exception propagating up the stack).  If
     *       any of the objects previously protected by these monitors were in
     *       an inconsistent state, the damaged objects become visible to
     *       other threads, potentially resulting in arbitrary behavior.  ...
     */
    @Deprecated(since="1.2")
    public final void stop() { 
```

stop 方法是不安全的。stop 方法停止线程会导致线程持有的锁（monitors）被释放，如果那些之前被锁住的对象处于不一致的状态，那么这些被损坏的对象变得对其他线程可见，导致异常结果。

意思就是原本应该被锁住的代码提前释放了，导致其他线程观察到的值不一致，出现异常结果。

如果想停止线程，可以使用 interrupt 方法中断线程，或者使用 boolean 标记位来判断是否继续执行 Thread 里面的逻辑。

## interrupt 方法中断线程

可以使用 interrupt 中断一个可能抛出 InterruptedException 的线程，比如 Thread.sleep。通过 isInterrupted 判断是否被中断了，如果没有被中断就循环执行，否则跳出循环。

### isInterrupted 判断

```java
public static void main(String[] args) throws InterruptedException {
        Runnable runnable = new Runnable() {

            @Override
            public void run() {
                while (!Thread.currentThread().isInterrupted()) {
                    try {
                        Thread.sleep(5000);
                        System.out.println("sleep 5");
                    } catch (InterruptedException e) {
                        System.out.println("interrupted:" + e);
                        e.printStackTrace();
                    }
                }
            }
        };
        Thread thread = new Thread(runnable);
        thread.start();

        TimeUnit.SECONDS.sleep(6);
        thread.interrupt();
    }
```

以上代码输出如下：

```
sleep 5
interrupted:java.lang.InterruptedException: sleep interrupted
java.lang.InterruptedException: sleep interrupted
	at java.base/java.lang.Thread.sleep(Native Method)
	at concurrent.TestThread$1.run(TestThread.java:13)
	at java.base/java.lang.Thread.run(Thread.java:835)
sleep 5
sleep 5
...
```


### 抛出 InterruptedException 异常中断复位

注意到以上代码 catch 到 InterruptedException 仍然继续执行了，这是因为抛出异常后中断标志位会复位，所以 isInterrupted 依然为 false，继续执行。

如果在 catch 语句中加上

```java
Thread.currentThread().interrupt();
```

那么线程就直到 interrupt 标记位为 true，停止执行。

```java
public static void main(String[] args) throws InterruptedException {
        Runnable runnable = new Runnable() {

            @Override
            public void run() {
                while (!Thread.currentThread().isInterrupted()) {
                    try {
                        Thread.sleep(5000);
                        System.out.println("sleep 5");
                    } catch (InterruptedException e) {
                        System.out.println("interrupted:" + e);
                        e.printStackTrace();
                        Thread.currentThread().interrupt();
                    }
                }
            }
        };
        Thread thread = new Thread(runnable);
        thread.start();

        TimeUnit.SECONDS.sleep(6);
        thread.interrupt();
    }
```

输出如下：

```
sleep 5
interrupted:java.lang.InterruptedException: sleep interrupted
java.lang.InterruptedException: sleep interrupted
	at java.base/java.lang.Thread.sleep(Native Method)
	at concurrent.TestThread$1.run(TestThread.java:13)
	at java.base/java.lang.Thread.run(Thread.java:835)

Process finished with exit code 0
```

### 不支持 interrupt 的场景

需要注意的是如果线程原本不支持 interrupt，那么它不会被 interrupt 中断影响。

```java
    public static void main(String[] args) throws InterruptedException {
        Runnable runnable = new Runnable() {

            @Override
            public void run() {
                int i = 0;
                while (i < 1000) {
                    System.out.println("i = " + i);
                    i++;
                }
            }
        };
        Thread thread = new Thread(runnable);
        thread.start();
        TimeUnit.MILLISECONDS.sleep(10);
        System.out.println("start interrupt");
        thread.interrupt();
    }
```

此时循环不会被中断，将一直执行。

## 安全地终止线程

除了 interrupt 方法，可以使用一个 volatile 修饰的 boolean 变量控制线程是否停止。

下面的代码使用了 volatile boolean 值表示是否继续循环，同时提供了一个 cancel 方法改变 on 的值来停止循环。

```java
    public static void main(String[] args) throws InterruptedException {
        MoonRunner runnable = new MoonRunner();
        Thread thread = new Thread(runnable);
        thread.start();
        TimeUnit.SECONDS.sleep(1);
        runnable.cancel();
    }

    public static class MoonRunner implements Runnable {
        private long i;
        private volatile boolean on = true;

        @Override
        public void run() {
            while (on) {
                i++;
                System.out.println("i=" + i);
            }
            System.out.println("stop");
        }

        public void cancel() {
            on = false;
        }
    }
```

输出如下：

```
...
i=219256
i=219257
stop
```

因为主线程 main 和 子线程 thread 都有可能改变 on 的值，为了保证可见性，使用 volatile 修饰 on，这样当有其他线程改变 on 的值时，所有的线程都会感知到它的变化。volatile 关键字可以保证线程可见性。