# Java 重入锁与条件对象

重入锁 ReentrantLock 是 Java 1.5 引入的，重入的意思是指可以重复获取锁，即拿到锁的对象可以再次拿一次锁，而不必先释放上一个锁。

ReentrantLock 实现了 Lock 接口。用 ReentrantLock 保护代码块的结构如下：

```java
    private void dosomethingLock() {
        Lock lock = new ReentrantLock();
        lock.lock();
        try {
            dosomething();
        } finally {
            lock.unlock();
        }
    }
```

这一结构保证了任何时候只有一个线程进入临界区，临界区就是在同一个时刻只有一个任务可以访问的代码区。unlock 操作放在 finally 可以保证锁一定可以被释放。

如果进入临界区时，发现在某一个条件满足之后才能执行，这时可以使用一个条件对象 Condition 来管理那些已经获得了锁但是却不能做有用工作的线程。

下面用一个转账的例子说明重入锁和条件对象。

```java
public class Alipay {
    private double[] accounts;
    private Lock alipayLock;

    public Alipay(int n, double money) {
        accounts = new double[n];
        alipayLock = new ReentrantLock();
        for (double a : accounts) {
            a = money;
        }
    }
}
```

以上代码定义了一个 Alipay 用来管理 n 个账户，其中有一个重入锁 alipayLock。

接着定义一个转账方法 transfer。

```java
    public void transfer(int from, int to, int amount) {
        alipayLock.lock();
        try {
            while (accounts[from] < amount) {
                condition.await();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            alipayLock.unlock();
        }
    }
```

如果转账的金额大于当前账户的余额，那么条件对象 condition 就 await 等待，直到其他账户先转入给它足够的余额。这里的 condition 是通过 ReentrantLock 的 newCondition 方法构造的。

调用了 condition 的 await 后就会放弃锁，同时等待，进入该 condition 的等待集。一旦有其他线程调用了同一个条件对象的 signalAll 方法后，就会重新激活因为这个条件等待的所有的线程。

```java
    public void transfer(int from, int to, int amount) {
        alipayLock.lock();
        try {
            while (accounts[from] < amount) {
                condition.await();
            }
            accounts[from] -= amount;
            accounts[to] += amount;
            condition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            alipayLock.unlock();
        }
    }
```

调用 signalAll 方法时并不是立即激活了一个等待线程，而是解除了等待线程的阻塞，让它们继续往下执行。还有一个方法是 signal，它是解除某一个线程的阻塞，如果该线程仍然不能运行，则再次被阻塞，如果没有其他线程再次调用 signal，那么系统就死锁了。

最终的代码如下，账户 1 依靠账户 0 的转账使得账户 1 有足够的余额转给账户 2。

```java
public class Alipay {
    private double[] accounts;
    private Lock alipayLock;
    private Condition condition;

    public Alipay(int n, double money) {
        accounts = new double[n];
        alipayLock = new ReentrantLock();
        condition = alipayLock.newCondition();
        for (int i = 0; i < n; ++i) {
            accounts[i] = money;
        }
    }

    public void transfer(int from, int to, int amount) {
        alipayLock.lock();
        try {
            while (accounts[from] < amount) {
                System.out.println("no enough, await: " + Thread.currentThread().getName());
                condition.await();
            }
            accounts[from] -= amount;
            accounts[to] += amount;
            condition.signalAll();
            System.out.println("transfer ok, signal all:" + Thread.currentThread().getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            alipayLock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Alipay alipay = new Alipay(3, 20);
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("transfer 1 to 2, amount 30");
                alipay.transfer(1, 2, 30);
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("transfer 0 to 1, amount 10");
                alipay.transfer(0, 1, 10);
            }
        }).start();
    }
}

```

以上代码输出结果如下：

```
transfer 1 to 2, amount 30
no enough, await: Thread-0
transfer 0 to 1, amount 10
transfer ok, signal all:Thread-1
transfer ok, signal all:Thread-0
```

可以看出账户 0 的转账调用了 signalAll，这时线程 Thread-0 收到信号继续执行，当它发现余额足够转出就跳出循环，往下执行。