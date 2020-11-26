# Java 线程的状态

Java 线程在运行的生命周期中可能处于 6 种不同的状态：

1. New 新创建的状态。线程被创建，但是没有调用 start 方法，在运行之前有一些基础工作要做，比如初始化操作。
2. Runnable 可运行状态。一旦调用了 start 方法，线程就处于 Runnable 状态。一个可运行的线程可能正在运行也可能没有运行，取决于操作系统给线程提供运行的时间。
3. Blocked 阻塞状态。表示线程被阻塞，暂时不活动。比如线程正在请求锁。
4. Waiting 等待状态。线程暂时不活动，并且不允许任何代码，这消耗最少的资源，知道线程调度器重新激活它。
5. Timed waiting 超时等待状态。和等待状态的区别是超时等待可以在指定的时间自行返回。
6. Terminated 终止状态。表示当前线程执行完毕。可能有 2 种情况：第一种情况是 run 方法正常执行完毕退出。第二种情况是因为一个没有被捕获的异常而终止了 run 方法，导致线程进入终止状态。

New 状态 Thread.start 后会进入 Runnable 状态。

Runnable 调用 Object.wait 或者 Thread.join 会进入 Waiting 状态。

Waiting 调用 Object.notify 或者 Object.notifyAll 会进入 Runnable 状态。

Runnable 调用 Thread.sleep(long)、Object.wait(long) 、 Thread.join(long) 会进入 Timed Waiting 状态。

Timed Waiting 调用 Object.notify 或者 Object.notifyAll 会进入 Runnable 状态。

Runnable 请求锁会进入 Blocked 状态。

Blocked 得到锁会进入 Runnable 状态。

Runnable 执行完毕或者异常退出会进入 Terminated 状态。
