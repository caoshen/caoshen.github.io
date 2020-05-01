# Java 线程 Runnable 与 Callable

如果想在新线程运行代码，可以使用 Thread 传递 Runnable 参数或者使用 Executor 调用 Callable 对象的方法。两者的区别主要在于 Callable 会返回一个 Future 对象，用来表示执行结果。

## Thread 和 Runnable

运行 Thread 通常有 2 种方法:

1. 继承 Thread 类，复写它的 run 方法。
2. 实现 Runnable 接口，作为 Thread 类的构造参数传递。

### 继承 Thread 类

自定义的 ExtendThread 继承 Thread 类，复写 run 方法，最后调用 start 启动线程。

```java
    public static class ExtendThread extends Thread {
        @Override
        public void run() {
            super.run();
            System.out.println("extend thread and override run");
        }
    }
    
    public static void main(String[] args) {
        new ExtendThread().start();
    }
```

注意这里有一个 super.run，它会调用 Thread 的 run 方法：

```java
    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
```

target 是 Thread 构造时传入的 runnable，因为构造 ExtendThread 时没有传递 runnable，所以不会调用到。

### 实现 Runnable 接口

可以直接构造一个匿名 Runnable，然后作为构造参数传递给 Thread 类，最后调用 start 方法启动线程。

```java
    public static void main(String[] args) {
        Runnable runnable = new Runnable() {
            public void run() {
                System.out.println("just run");
            }
        };
        new Thread(runnable).start();
    }
```

这里比较推荐第二种方式，即实现 Runnable 接口优于继承 Thread 类（接口优于继承）。因为代码的主要逻辑都在 run 方法中，如果有新的任务需要在 Thread 执行，只需要替换 Runnable 即可，不用关心 Thread 的内部实现。自定义类继承 Thread 的方式重写了 run 方法，破坏了 Thread 的原有结构，如果有新的任务要执行就又要构造一个自定义 Thread。

## Callable 和 Future

Callable 是 java.util.concurrent 的一个接口类。

```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

可以看出 Callable 提供了返回值，或者抛出一个异常，而且支持泛型。

Callable 通常和 Executors 里面的线程池配合使用，返回一个 Future 类型表示执行结果。Future 的 get 方法用来获取执行结果，但是 get 会阻塞线程，直到 call 方法执行完毕。

```java
    public static void main(String[] args) {
        Callable<String> callable = new Callable<String>() {
            public String call() throws Exception {
                return "hello callable";
            }
        };
        Future<String> future = Executors.newSingleThreadExecutor().submit(callable);
        try {
            System.out.println(future.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
```

## CompletableFuture

假设有 2 个 Callable 分别在执行，它们执行结束的时间可能不同，如果想要等待 2 者都执行完毕后再做一些事情，可以使用 CompletableFuture。

CompletableFuture 类实现了 Future 和 CompletionStage 接口。

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T>
```

supplyAsync 方法可以返回一个在 ForkJoinPool#commonPool() 执行的 CompletableFuture。

thenCombine 方法会把 2 个 CompletableFuture 联合执行，并且把它们的执行结果传递给一个二元方法，得到一个新的 CompletableFuture。

join 方法得到最终的执行结果。

```java
    public static void main(String[] args) {
        CompletableFuture<String> supply1 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "hello3";
        });
        CompletableFuture<String> supply2 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "hello5";
        });
        String result = supply1.thenCombine(supply2, (s, s2) -> s + s2).join();
        System.out.println("result:" + result);
    }
 ```

 5 秒后输出结果如下:

 ```
result:hello3hello5
 ```