---
title: OkHttp 源码解析
date: 2019-08-20 00:41:10
tags: Android
categories: Android
---

# OkHttp 源码解析

本文主要从两个方面分析 OkHttp 源代码。
1. 网络请求流程；
2. 连接池复用。

## OkHttp 的网络请求流程

### 网络请求流程

不管是异步请求还是同步请求，都会先通过 OkHttpClient 的 newCall 方法构造一个 Call。

```kotlin
  /** Prepares the [request] to be executed at some point in the future. */
  override fun newCall(request: Request): Call {
    return RealCall.newRealCall(this, request, forWebSocket = false)
  }
```

可以看出 newCall 方法返回的是 RealCall。不管是异步的 enqueue 方法还是同步的 execute 方法，都是由 RealCall 调用的。

enqueue 代码如下：

```kotlin
  override fun enqueue(responseCallback: Callback) {
    synchronized(this) {
      check(!executed) { "Already Executed" }
      executed = true
    }
    transmitter.callStart()
    client.dispatcher.enqueue(AsyncCall(responseCallback))
  }
```

execute 代码如下：

```kotlin
  override fun execute(): Response {
    synchronized(this) {
      check(!executed) { "Already Executed" }
      executed = true
    }
    transmitter.timeoutEnter()
    transmitter.callStart()
    try {
      client.dispatcher.executed(this)
      return getResponseWithInterceptorChain()
    } finally {
      client.dispatcher.finished(this)
    }
  }
```

可以发现，在 RealCall 中 enqueue 和 execute 最终的请求是 dispatcher 调用的。

### Dispatcher 任务调度

Dispatcher 类定义了控制并发的请求，包括最大请求数、消费者线程池、同步和异步的请求队列等变量。

```kotlin
  @get:Synchronized var maxRequests = 64

  @get:Synchronized var maxRequestsPerHost = 5

  @get:JvmName("executorService") val executorService: ExecutorService

  /** Ready async calls in the order they'll be run. */
  private val readyAsyncCalls = ArrayDeque<AsyncCall>()

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private val runningAsyncCalls = ArrayDeque<AsyncCall>()

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private val runningSyncCalls = ArrayDeque<RealCall>()

```

maxRequests 最大请求数

maxRequestsPerHost 每个主机的最大请求数

ExecutorService 消费者线程池

readyAsyncCalls 将要运行的异步请求队列

runningAsyncCalls 正在运行的异步请求队列

runningSyncCalls 正在运行的同步请求队列

```kotlin
  @get:Synchronized
  @get:JvmName("executorService") val executorService: ExecutorService
    get() {
      if (executorServiceOrNull == null) {
        executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
            SynchronousQueue(), threadFactory("OkHttp Dispatcher", false))
      }
      return executorServiceOrNull!!
    }
```

executorService 的 get 方法返回一个 ThreadPoolExecutor，它的核心线程数是 0，最大线程数是 Int.MAX_VALUE，线程等待时间是 60 秒，任务队列是 SynchronousQueue。

之前说过 RealCall 的 enqueue 和 execute 实际上是调用的 Dispatcher 的 enqueue 和 execute 。

```kotlin
  internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
      readyAsyncCalls.add(call)

      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
      // the same host.
      if (!call.get().forWebSocket) {
        val existingCall = findExistingCallWithHost(call.host())
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
      }
    }
    promoteAndExecute()
  }
```

enqueue 方法主要由以下 3 步：

1. Dispatcher 的 enqueue 方法先将 AsyncCall 加入到 readyAsyncCalls 队列。

2. 如果入队的 call 不是 websocket，那么查找任务准备队列（readyAsyncCalls）和任务执行队列（runningAsyncCalls），看里面是不是存在相同主机（host）的请求。如果存在，就把当前的 call 的主机访问数量设为已经存在的相同主机的 call 数量。这个操作保证每个主机的请求数不超过设定值。在后面的 promoteAndExecute 会被用到。

3. promoteAndExecute 方法会从 readyAsyncCalls 队列中筛选出符合条件的 call，然后调用 call 的 executeOn(executorService) 执行。

promoteAndExecute 代码如下：

```kotlin
  private fun promoteAndExecute(): Boolean {
    assert(!Thread.holdsLock(this))

    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
      val i = readyAsyncCalls.iterator()
      while (i.hasNext()) {
        val asyncCall = i.next()

        if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
        if (asyncCall.callsPerHost().get() >= this.maxRequestsPerHost) continue // Host max capacity.

        i.remove()
        asyncCall.callsPerHost().incrementAndGet()
        executableCalls.add(asyncCall)
        runningAsyncCalls.add(asyncCall)
      }
      isRunning = runningCallsCount() > 0
    }

    for (i in 0 until executableCalls.size) {
      val asyncCall = executableCalls[i]
      asyncCall.executeOn(executorService)
    }

    return isRunning
  }
```

RealCall 的 executeOn 方法有个 success 局部变量。正常情况下，execute 走完，success 是 true，但是如果这个过程中抛出异常，那么 success 就还是 false。在最后的 finally 语句中可以发现，会调用 Dispatcher 的 finish 方法。finish 方法其实就是从 runningAsyncCalls 队列移除 call，然后再次调用 promoteAndExecute 方法。

```kotlin
    /**
     * Attempt to enqueue this async call on [executorService]. This will attempt to clean up
     * if the executor has been shut down by reporting the call as failed.
     */
    fun executeOn(executorService: ExecutorService) {
      assert(!Thread.holdsLock(client.dispatcher))
      var success = false
      try {
        executorService.execute(this)
        success = true
      } catch (e: RejectedExecutionException) {
        val ioException = InterruptedIOException("executor rejected")
        ioException.initCause(e)
        transmitter.noMoreExchanges(ioException)
        responseCallback.onFailure(this@RealCall, ioException)
      } finally {
        if (!success) {
          client.dispatcher.finished(this) // This call is no longer running!
        }
      }
    }
```

<!-- ### Interceptor 拦截器

### 缓存策略

### 失败重连

## OkHttp 的复用连接池 -->
