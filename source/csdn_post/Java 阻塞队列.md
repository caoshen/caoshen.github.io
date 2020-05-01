# Java 阻塞队列

阻塞队列在 Java 中时常被用到，比如线程池、生产者消费者模型。

## 阻塞队列简介

阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

### 常见阻塞场景

阻塞队列有两个常见的阻塞场景，它们分别是：

1. 当队列中没有数据的情况下，消费者端的所有线程都会被自动阻塞（挂起），直到有数据放入队列。

2. 当队列中填满了数据的情况下，生产者端的所有线程都会被自动阻塞（挂起），直到队列中有空的位置，线程自动被唤醒。

支持以上两种阻塞场景的队列被称为阻塞队列。

### BlockingQueue 的核心方法

BlockingQueue 的核心方法可以分为放入数据和读取数据。

放入数据：

- boolean offer(E e) 表示如果可能的话，将 e 加到 BlockingQueue 里。如果 BlockingQueue 可以容纳，则返回 true，否则返回 false。本方法不阻塞当前执行方法的线程。

- boolean offer(E e, long timeout, TimeUnit unit) 可以设定等待的时间。如果在指定的时间内还不能往队列中加入 BlockingQueue，则返回失败。

- void put(E e) 将 e 加到 BlockingQueue 里。如果 BlockingQueue 没有空间，则调用此方法的线程被阻断，直到 BlockingQueue 里面有空间在继续。


读取数据：

- E poll() 取走 BlockingQueue 里排在首位的对象。

- E poll(long timeout, TimeUnit unit) 从 BlockingQueue 中取出一个队首的对象。如果在指定时间内队列有数据可取，则立即返回队列中的数据，否则直到时间超时还没有数据可取，就返回失败。

- E take() 取走 BlockingQueue 里排在首位的对象。如果 BlockingQueue 是空，则阻断进入等待状态，否则返回失败。

- int drainTo(Collection<? super E> c) 一次性从 BlockingQueue 获取所有可用的数据对象（也可以指定取出的个数）。通过该方法可以提升获取数据的效率，无需多次分批加锁或者释放锁。注意这里的下界通配符 super，表示 c 集合中的元素必须至少是 E 类型或者它的超类。

## Java 中的阻塞队列

Java 中提供了很多阻塞队列，举例如下：

- ArrayBlockingQueue 由数组结构组成的有界阻塞队列
- LinkedBlockingQueue 由链表结构组成的有界阻塞队列
- PriorityBlockingQueue 支持优先级排序的无界阻塞队列
- DelayQueue 使用优先级队列实现的无界阻塞队列
- SynchronousQueue 不存储元素的阻塞队列
- LinkedTransferQueue 由链表结构组成的无界阻塞队列
- LinkedBlokcingDeque 由链表结构组成的阻塞双端队列

这些阻塞队列有一个共同点，就是都实现了 BlockingQueue 接口。

### ArrayBlockingQueue

ArrayBlockingQueue 是用数组实现的有界阻塞队列，按照先进先出 FIFO 的原则对元素进行排序。默认情况下不保证线程公平地访问队列。公平访问队列就是指阻塞地所有生产者线程或者消费者线程按照阻塞地先后顺序访问队列。即先阻塞地生产者线程，可以先往队列里面插入元素；先阻塞的消费者线程，可以先从队列里获取元素。通常情况下，保证公平性会降低吞吐量。

我们可以使用以下代码创建一个公平的阻塞队列：

```java
    ArrayBlockingQueue fairQueue = new ArrayBlockingQueue(2000, true); 
```

ArrayBlockingQueue 的第二个参数传 true，保证公平性。

### LinkedBlockingQueue

LinkedBlockingQueue 是基于链表的阻塞队列，同 ArrayListBlockingQueue 类似，此队列按照先进先出（FIFO）的原则对元素进行排序，其内部也维持着一个数据缓存队列（该队列由一个链表构成）。当生产者往队列中放入一个数据时，队列会从生产者手中获取数据，并缓存在队列内部，而生产者立即返回。当队列缓冲区达到缓存容量的最大值时（LinkedBlockingQueue 可以通过构造方法指定该值），才会阻塞生产者队列，直到消费者从队列中消费掉一份数据，生产者线程会被唤醒。反之，对于消费者这端的处理也基于同样的原理。

LinkedBlockingQueue 之所以能够高效地处理并发数据，因为它对于生产者端和消费者端分别采用了独立地锁来控制数据同步。这也意味着在高并发地情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。

作为开发者需要注意的是，如果构造 LinkedBlockingQueue 时，没有指定大小，那么默认会用最大整数（Integer.MAX_VALUE）作为容量。如果生产者的速度大于消费者的速度，也许还没有等到队列满阻塞产生，系统内存就已经被消耗殆尽了。

ArrayBlockingQueue 和 LinkedBlockingQueue 是两个最普通也是最常用的阻塞队列。一般情况下，在处理多线程的生产者消费者问题时，使用这两个类就行。

### PriorityBlockingQueue

PriorityBlockingQueue 是一个支持优先级的无界队列。默认情况下元素采取自然顺序升序排列。这里可以自定义元素的 compareTo 方法得到排序规则。也可以在构造 PriorityBlockingQueue 时传入 Comparator 来对元素进行排序，但是不能保证同优先级元素的顺序。

### DelayQueue

DelayQueue 是一个支持延时获取元素的无界阻塞队列。队列使用 PriorityQueue 来实现。队列中的元素必须实现 Delayed 接口。创建元素时，可以指定元素到期的时间，只有在元素到期时才能从队列中取走。

### SynchronousQueue

SynchronousQueue 是一个不存储元素的阻塞队列。每个插入操作必须等待另一个线程的移除操作，同样任何一个移除操作都等待另一个线程的插入操作。因此此队列内部其实并没有任何一个元素，或者说容量是 0，严格来说它不是一种容器。由于队列没有容量，它的 peek 操作（返回队列的头元素）总是返回 null。

```java
    public E peek() {
        return null;
    }
```

### LinkedTransferQueue

LinkedTransferQueue 是一个由链表结构组成的无界阻塞 Transfer 队列。LinkedTransferQueue 实现了一个重要的接口 TransferQueue。该接口有 5 个方法，其中有 3 个重要的方法，它们分别如下所示。

1. void transfer(E e) 若当前存在一个正在等待获取的消费者线程，则立即将元素传递给消费者；如果没有消费者在等待接收数据，就会将元素插入到队尾底部，并且等待进入阻塞状态，直到有消费者线程取走该元素。

2. boolean tryTransfer(E e) 若当前存在一个正在等待获取的消费者线程，则立即将元素传递给消费者；若不存在，则返回 false，并且不进入队列，这是一个不阻塞的操作。与 transfer 方法不同的是，tryTransfer 方法无论消费者是否接收，其都会立即返回；而 transfer 方法则是消费者接收了才返回。

3. boolean tryTransfer(E e, long timeout, TimeUnit unit) 若当前存在一个正在等待获取的消费者线程，则立即将元素传递给消费者；若不存在则将元素插入到队列尾部，并且等待消费者线程取走该元素。若在指定的超时时间内元素未被消费者线程获取，则返回 false；若在指定的超时时间内被消费者线程获取，则返回 true。

### LinkedBlockingDeque

LinkedBlockingDeque 是由一个链表结构组成的双向阻塞队列。双向队列可以从队列的两端插入和移除元素，因此在多线程同时入队时，减少了一半的竞争。由于是双向的，因此 LinkedBlockingDeque 多了 addFirst、addLast、offerFirst、offerLast、peekFirst、peekLast 等方法。以 First 开头的表示插入、获取、移除双端队列的第一个元素；以 Last 结尾的方法表示插入、获取、移除双端队列的最后一个元素。

## 阻塞队列的实现原理

这里以 ArrayBlockingQueue 为例分析它的方法实现。

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    private static final long serialVersionUID = -817911632652898426L;

    final Object[] items;

    int takeIndex;

    int putIndex;

    int count;

    final ReentrantLock lock;

    private final Condition notEmpty;

    private final Condition notFull;

    transient Itrs itrs;
    ...
```

可以看出 ArrayBlockingQueue 维护了一个 Object 类型的数组，takeIndex 和 putIndex 分别表示队首元素和队尾元素的下标。count 表示队列中元素的个数，lock 则是一个可重入锁，notEmpty 和 notFull 是等待条件。

ArrayBlockingQueue 的 put 操作如下：

```java
    public void put(E e) throws InterruptedException {
        Objects.requireNonNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```

从 put 方法可以看出，它先获取了 lock 锁，并且调用了 lockInterruptibly 方法表示可中断，然后循环判断当前元素个数是否等于数组的长度，如果相等就调用 notFull.await 进行等待。当此线程被其他线程唤醒时，通过 enqueue 方法插入一个元素，最后解锁。

ArrayBlockingQueue 的 enqueue 方法如下：

```java
    private void enqueue(E e) {
        // assert lock.isHeldByCurrentThread();
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = e;
        if (++putIndex == items.length) putIndex = 0;
        count++;
        notEmpty.signal();
    }
```

可以看出 enqueue 方法把 putIndex 位置的元素赋值为 e。如果已经是最后一个位置，那么会重置为 0，然后元素个数加 1，最后调用条件对象 notEmpty 的 signal 方法，表示唤醒正在等待取元素的线程。

ArrayBlockingQueue 的 take 方法如下：

```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```

和 put 方法类似，take 方法等待的是 notEmpty 条件。在 take 方法中，如果元素个数为 0，就循环等待。否则调用 dequeue 方法出队一个元素。

ArrayBlockingQueue 的 dequeue 方法如下：

```java
    private E dequeue() {
        final Object[] items = this.items;
        E e = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length) takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return e;
    }
```

可以看出在得到 takeIndex 位置的元素后会调用 notFull 条件的 signal 方法，表示唤醒正在等待插入元素的线程。

## 阻塞队列的使用场景

除了线程池的实现使用了阻塞队列之外，还可以在生产者消费者模式中使用阻塞队列。

首先使用 Object.wait 、Object.notify 和非阻塞队列 PriorityQueue 实现生产者消费者模式。

```java
package concurrent;

import java.util.PriorityQueue;

public class QueueTest {
    private int queueSize = 10;
    private PriorityQueue<Integer> queue = new PriorityQueue<>(queueSize);

    public static void main(String[] args) {
        QueueTest queueTest = new QueueTest();
        Producer producer = queueTest.new Producer();
        Consumer consumer = queueTest.new Consumer();
        producer.start();
        consumer.start();
    }

    class Consumer extends Thread {
        @Override
        public void run() {
            while (true) {
                synchronized (queue) {
                    while (queue.size() == 0) {
                        System.out.println("队列空，等待数据" + queue);
                        try {
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    // 消费队首元素
                    Integer poll = queue.poll();
                    System.out.println("队列取走" + poll + ", " + queue);
                    queue.notify();
                }
            }
        }
    }

    class Producer extends Thread {
        @Override
        public void run() {
            while (true) {
                synchronized (queue) {
                    while (queue.size() == queueSize) {
                        System.out.println("队列满，等待有空余空间" + queue);
                        try {
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    // 插入一个元素
                    System.out.println("队列插入 1" + queue);
                    queue.offer(1);
                    queue.notify();
                }
            }
        }
    }
}
```

然后使用阻塞队列 ArrayBlockingQueue 实现生产者消费者模式。

```java
package concurrent;

import java.util.concurrent.ArrayBlockingQueue;

public class BlockingQueueTest {
    private int queueSize = 10;
    private ArrayBlockingQueue<Integer> queue = new ArrayBlockingQueue<>(queueSize);

    public static void main(String[] args) {
        BlockingQueueTest queueTest = new BlockingQueueTest();
        BlockingQueueTest.Producer producer = queueTest.new Producer();
        BlockingQueueTest.Consumer consumer = queueTest.new Consumer();
        producer.start();
        consumer.start();
    }

    class Consumer extends Thread {
        @Override
        public void run() {
            while (true) {
                try {
                    // 消费队首元素
                    Integer poll = queue.take();
                    System.out.println("队列取走" + poll + ", " + queue);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    class Producer extends Thread {
        @Override
        public void run() {
            while (true) {
                try {
                    // 插入一个元素
                    System.out.println("队列插入 1" + queue);
                    queue.put(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

可以看出使用阻塞队列实现无须单独考虑同步和线程间通信的问题，实现起来比较简单。