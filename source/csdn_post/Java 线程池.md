# Java 线程池

线程的创建和消耗都需要一定的开销，如果每次执行一个异步任务都需要开一个线程去执行，则这些线程的创建和销毁将消耗大量的资源，而且线程都是各自为政，很难对其进行控制，更何况有一堆的线程在执行。这时就需要线程池对线程进行管理。

在 Java 1.5 中提供了 Executor 框架用于把任务的提交和执行解耦，任务的提交交给 Runnable 或者 Callable，而 Executor 框架用来处理任务。Executor 框架中最核心的成员就是 ThreadPoolExecutor，它是线程池的核心实现类

## ThreadPoolExecutor

可以通过 ThreadPoolExecutor 来创建一个线程池，ThreadPoolExecutor 类一共有 4 个构造方法。其中，拥有最多参数的构造方法如下所示：

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

共有 7 个参数，这些参数的作用如下：

- corePoolSize 核心线程数。默认情况下线程池是空的，只有任务提交时才会创建线程。如果当前运行的线程数少于核心线程数，则创建新的线程来处理任务。如果等于或者多于核心线程数，则不再创建。如果调用线程池的 prestartAllCoreThreads 方法，线程池会提前创建并且启动所有的核心线程来等待任务。

- maximumPoolSize 线程池允许创建的最大线程数。如果任务队列满了并且线程数小于 maximumPoolSize，则线程池仍旧会创建新的线程来处理任务。

- keepAliveTime 非核心线程闲置的超时时间。如果超过这个时间就回收线程。如果任务很多，并且每个任务的执行时间很短，则可以调大 keepAliveTime 来提高线程的利用率。另外，如果设置 allowCoreThreadTimeout 为 true （通过 allowsCoreThreadTimeOut 方法），keepAliveTime 也会应用到核心线程上。

- TimeUnit keepAliveTime 参数的时间单位。可选单位有天、小时、分钟、秒、毫秒等。

- workQueue 任务队列。如果当前任务数大于核心线程数，则将任务添加到任务队列。这里的任务队列是阻塞队列（BlockingQueue）。

- ThreadFactory 线程工厂。可以用线程工厂来给每个创建出来的线程设置名字。如果不设置会使用默认的线程工厂。

- RejectedExecutionHandler 饱和策略。这是当任务队列和线程池都满了时所采取的应对策略，默认是 AbortPolicy，表示无法处理新任务，并抛出 RejectedExecutionException 异常。除此之外还有 3 中策略，它们分别如下：

1. CallerRunsPolicy 用调用者所在的线程处理任务。此任务提供简单的反馈控制机制，能够减缓新任务的提交速度。

2. DiscardPolicy 不能执行的任务，并将该任务删除。

3. DiscardOldestPolicy 丢弃队列最近的任务，并执行当前的任务。

## 线程池的处理流程和原理

线程池的处理流程主要可以分为 3 个步骤，如下所示：

1. 提交任务后，线程池先判断线程数是否达到了核心线程数 corePoolSize。如果未达到核心线程数，则创建核心线程处理任务。否则就执行下一步操作。

2. 接着线程池判断任务队列是否满了。如果没满，则将任务添加到任务队列中；否则，就执行下一步操作。

3. 如果任务队列满了，线程池就判断线程数是否达到了最大线程数。如果未达到，则创建非核心线程处理任务；否则就执行饱和策略，默认会抛出 RejectedExecutionException 异常。


如果我们执行 ThreadPoolExecutor 的 execute 方法，会遇到各种情况：

1. 如果线程池中的线程数未达到核心线程数，则创建核心线程处理任务。

2. 如果线程数大于或者等于核心线程数，则将任务加入到任务队列，线程池中的空闲线程会不断地从任务队列中取出任务进行处理。

3. 如果任务队列满了，并且线程数没有达到最大线程数，则创建非核心线程去处理任务。

4. 如果线程数超过了最大线程数，则执行饱和策略。

## 线程池的种类

通过直接或者间接地配置 ThreadPoolExecutor 的参数可以创建不同类型的 ThreadPoolExecutor，其中有几种线程池比较常用，它们分别是 FixedThreadPool、CachedThreadPool、SingleThreadExecutor、ScheduledThreadPool、ForkJoinPool。

- FixedThreadPool

FixedThreadPool 是可重用固定线程数的线程池。在 Executors 类提供了创建 FixedThreadPool 的方法 newFixedThreadPool，如下所示：

```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

FixedThreadPool 的 corePoolSize 和 maximumPoolSize 都设置为创建 FixedThreadPool 指定的参数 nThreads，也就是说 FixedThreadPool 只有核心线程，并且数量是固定的，没有非核心线程。keekAliveTime 设置为 0L 意味者多余的线程会被立即终止。因为不会产生多余的线程，所以 keepAliveTime 是无效的参数。另外，任务队列采用了无界的阻塞队列 LinkedBlockingQueue。

当执行 FixedThreadPool 的 execute 方法时，如果当前运行的线程未达到 corePoolSize （核心线程数）时就创建核心线程来处理任务，如果达到了核心线程数则将任务添加到 LinkedBlockingQueue 中。FixedThreadPool 就是一个有固定数量核心线程的线程池，并且这些核心线程不会被回收。当线程数超过 corePoolSize 时，就将任务存储在任务队列中；当线程池有空闲线程时，则从任务队列中去取任务执行。

- CachedThreadPool

CachedThreadPool 是一个根据需要创建线程的线程池，创建 CachedThreadPool 的代码如下：

```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

CachedThreadPool 的 corePoolSize 是 0，maximumPoolSize 设置为 Integer.MAX_VALUE，这意味着 CachedThreadPool 没有核心线程，非核心线程是无界的。keepAliveTime 设置为 60L，则空闲线程等待新任务的最长时间为 60s。在此用了阻塞队列 SynchronousQueue，它是一个不存储元素的阻塞队列，每个插入操作必须等待另一个线程的移除操作，同样任何一个移除操作都等待另一个线程的插入操作。

当执行 execute 方法时，首先会执行 SynchronousQueue 的 offer 方法来提交任务，并且查询线程池中是否有空闲的线程执行 SynchronousQueue 的 poll 方法来移除任务。如果有则配对成功，将任务交给这个空闲的线程处理；如果没有则配对失败，创建新的线程处理任务。当线程池中的线程空闲时，它会执行 SynchronousQueue 的 poll 方法，等待 SynchronousQueue 中新提交的任务。如果超过 60s 没有新任务提交到 SynchronousQueue，这个空闲线程将终止。

因为 maximumPoolSize 是无界的，所以如果提交的任务大于线程池中线程的处理速度，就会不断地创建新线程。另外，每次提交任务都会立即有线程去处理。所以，CachedThreadPool 比较适合有大量任务处理，但是每个任务耗时较少地情况。

- SingleThreadExecutor

SingleThreadExecutor 的 corePoolSize 和 maximumPoolSize 都为 1，意味着 SingleThreadExecutor 只有一个核心线程，其他参数和 FixedThreadPool 一样。

但是 SingleThreadExecutor 使用 FinalizableDelegatedExecutorService 封装了一次，在封装类中只暴露了 ExecutorService 接口的方法，这意味着 ThreadPoolExecutor 里面的 public 方法对 SingleThreadExecutor 来说是无法调用的。也就是说 SingleThreadExecutor 是无法配置线程池的各个参数的，但是 FixedThreadPool 和 ThreadPoolExecutor 可以自由配置。

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

当执行 execute 方法时，如果当前运行的线程数未达到核心线程数，也就是当前没有运行的线程，则创建一个新线程来处理任务。如果当前有运行的线程，则将任务添加到阻塞队列 LinkedBlockingQueue 中。因此，SingleThreadExecutor 能确保所有的任务在一个线程中按照顺序执行。

- ScheduledThreadPool

ScheduledThreadPool 是一个能实现定时和周期性任务的线程池。

```java
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```

ScheduledThreadPoolExecutor 的构造方法如下：

```java
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }
```

ScheduledThreadPoolExecutor 继承了 ThreadPoolExecutor 并且实现了 ScheduledExecutorService 接口。它主要用于给定延时之后的运行任务或者定期处理任务。

ScheduledThreadPoolExecutor 的构造方法调用了 ThreadPoolExecutor 的构造方法。corePoolSize 是传进来固定的数值，maximumPoolSize 的值是 Integer.MAX_VALUE。因 为 DelayedWorkQueue 是无界的，所以 Integer.MAX_VALUE 是无效的，总是用核心线程处理阻塞队列的任务。

当执行 ScheduledThreadPoolExecutor 的 scheduleAtFixedRate 或者 scheduleWithFixedDelay 方法时，会向 DelayedWorkQueue 添加一个 ScheduledFutureTask，并检查运行的线程是否达到 corePoolSize。如果没有则创建新线程并启动它，但是并不是立即去执行任务，而是去 DelayedWorkQueue 中取 ScheduledFutureTask，然后去执行任务。如果运行的线程达到了 corePoolSize 时，则将任务添加到 DelayedWorkQueue 中。DelayedWorkQueue 会将任务进行排序，先要执行的任务放在队列的前面。与此前介绍的线程池不同的是，当执行完任务后，会将 ScheduledFutureTask 中的 time 变量改为下次要执行的时间，并且放回到 DelayedWorkQueue 中。

- ForkJoinPool

Executors 中的 newWorkStealingPool 方法提供了工作窃取线程池，它是一个 ForkJoinPool。

```java
    public static ExecutorService newWorkStealingPool() {
        return new ForkJoinPool
            (Runtime.getRuntime().availableProcessors(),
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }
```

可以看出 newWorkStealingPool 构造了一个与 CPU 处理器核数相同的 ForkJoinPool，它适合多核并行计算的场景。

与其他线程池不同，ForkJoinPool 没有使用或者继承 ThreadPoolExecutor。它继承了 AbstractExecutorService。

```java
public class ForkJoinPool extends AbstractExecutorService {
```

java.util.concurrent 包中的 CompletableFuture 就用到了 ForkJoinPool。

```java
    private static final boolean USE_COMMON_POOL =
        (ForkJoinPool.getCommonPoolParallelism() > 1);

    /**
     * Default executor -- ForkJoinPool.commonPool() unless it cannot
     * support parallelism.
     */
    private static final Executor ASYNC_POOL = USE_COMMON_POOL ?
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();

```

可以看出如果机器 CPU 支持并行计算就会使用 ForkJoinPool。


