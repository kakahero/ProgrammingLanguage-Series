# Java 并发编程中线程池的使用

提交给线程池的 task 最好是同一种类的 task，因为不同种类的 task 执行的时间可能会有差异，这样的话如果线程池的大小是固定的，那么就会出现执行时间不均匀的情况。

task 中的 threadlocal 最好不要跨任务使用，因为线程池中的 worker 线程可能会随着任务的数量或者任务执行的异常增加或减少，这样跨任务的 threadlocal 传递就会出现问题。

并发 API 引入了 ExecutorService 作为一个在程序中直接使用 Thread 的高层次的替换方案。Executos 支持运行异步任务，通常管理一个线程池，这样一来我们就不需要手动去创建新的线程。在不断地处理任务的过程中，线程池内部线程将会得到复用，因此，在我们可以使用一个 executor service 来运行和我们想在我们整个程序中执行的一样多的并发任务。 下面是使用 executors 的第一个代码示例：

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
executor.submit(() -> {
	String threadName = Thread.currentThread().getName();
	System.out.println("Hello " + threadName);
});
// => Hello pool-1-thread-1
```

Executors 类提供了便利的工厂方法来创建不同类型的 executor services。在这个示例中我们使用了一个单线程线程池的 executor。 代码运行的结果类似于上一个示例，但是当运行代码时，你会注意到一个很大的差别：Java 进程从没有停止！Executors 必须显式的停止-否则它们将持续监听新的任务。 ExecutorService 提供了两个方法来达到这个目的——shutdwon()会等待正在执行的任务执行完而 shutdownNow()会终止所有正在执行的任务并立即关闭 execuotr。

```java
try {
    System.out.println("attempt to shutdown executor");
    executor.shutdown();
    executor.awaitTermination(5, TimeUnit.SECONDS);
    }
catch (InterruptedException e) {
    System.err.println("tasks interrupted");
}
finally {
    if (!executor.isTerminated()) {
        System.err.println("cancel non-finished tasks");
    }
    executor.shutdownNow();
    System.out.println("shutdown finished");
}
```

executor 通过等待指定的时间让当前执行的任务终止来“温柔的”关闭 executor。在等待最长 5 分钟的时间后，execuote 最终会通过中断所有的正在执行的任务关闭。

### invokeAll | 调用所有的 Callable

Executors 支持通过 invokeAll()一次批量提交多个 callable。这个方法结果一个 callable 的集合，然后返回一个 future 的列表。

```java
ExecutorService executor = Executors.newWorkStealingPool();

List<Callable<String>> callables = Arrays.asList(
        () -> "task1",
        () -> "task2",
        () -> "task3");

executor.invokeAll(callables)
    .stream()
    .map(future -> {
        try {
            return future.get();
        }
        catch (Exception e) {
            throw new IllegalStateException(e);
        }
    })
    .forEach(System.out::println);
```

在这个例子中，我们利用 Java8 中的函数流(stream)来处理 invokeAll()调用返回的所有 future。我们首先将每一个 future 映射到它的返回值，然后将每个值打印到控制台。如果你还不属性 stream，可以阅读我的[Java8 Stream 教程](http://winterbe.com/posts/2014/07/31/java8-stream-tutorial-examples/)。

### invokeAny

批量提交 callable 的另一种方式就是 invokeAny()，它的工作方式与 invokeAll()稍有不同。在等待 future 对象的过程中，这个方法将会阻塞直到第一个 callable 中止然后返回这一个 callable 的结果。 为了测试这种行为，我们利用这个帮助方法来模拟不同执行时间的 callable。这个方法返回一个 callable，这个 callable 休眠指定 的时间直到返回给定的结果。

```java
//这个callable方法是用来构造不同的Callable对象
Callable<String> callable(String result, long sleepSeconds) {
    return () -> {
        TimeUnit.SECONDS.sleep(sleepSeconds);
        return result;
    };
}
```

我们利用这个方法创建一组 callable，这些 callable 拥有不同的执行时间，从 1 分钟到 3 分钟。通过 invokeAny()将这些 callable 提交给一个 executor，返回最快的 callable 的字符串结果-在这个例子中为任务 2：

```java
ExecutorService executor = Executors.newWorkStealingPool();

List<Callable<String>> callables = Arrays.asList(
callable("task1", 2),
callable("task2", 1),
callable("task3", 3));

String result = executor.invokeAny(callables);
System.out.println(result);

// => task2
```

### Scheduled Executors

为了持续的多次执行常见的任务，我们可以利用调度线程池 ScheduledExecutorService 支持任务调度，持续执行或者延迟一段时间后执行：

```java
ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

Runnable task = () -> System.out.println("Scheduling: " + System.nanoTime());
// 设置延迟 3 秒执行
ScheduledFuture<?> future = executor.schedule(task, 3, TimeUnit.SECONDS);

TimeUnit.MILLISECONDS.sleep(1337);

long remainingDelay = future.getDelay(TimeUnit.MILLISECONDS);
System.out.printf("Remaining Delay: %sms", remainingDelay);
```

调度一个任务将会产生一个专门的 ScheduleFuture 类型，它除了提供了 Future 的所有方法之外，他还提供了 getDelay()方法来获得剩余的延迟。在延迟消逝后，任务将会并发执行。为了调度任务持续的执行，executors 提供了两个方法 s`cheduleAtFixedRate()` 和 `scheduleWithFixedDelay()`；第一个方法用来以固定频率来执行一个任务，比如，下面这个示例中，每分钟一次：

```java
ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

Runnable task = () -> System.out.println("Scheduling: " + System.nanoTime());

int initialDelay = 0;
int period = 1;
executor.scheduleAtFixedRate(task, initialDelay, period, TimeUnit.SECONDS);
```

另外，这个方法还接收一个初始化延迟，用来指定这个任务首次被执行等待的时长。需要注意的是，`scheduleAtFixedRate()` 并不考虑任务的实际用时。所以，如果你指定了一个 period 为 1 分钟而任务需要执行 2 分钟，那么线程池为了性能会更快的执行。在这种情况下，你应该考虑使用 scheduleWithFixedDelay()。这个方法的工作方式与上我们上面描述的类似。不同之处在于等待时间 period 的应用是在一次任务的结束和下一个任务的开始之间。

# 线程池

# 自定义线程池

```java
 public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}

ExecutorService executor = Executors.newFixedThreadPool(20);

return new ThreadPoolExecutor(20, 20,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
```

在需要大量自定义配置的情况下，ThreadPoolExecutor 为更为灵活。

```java
ThreadPoolExecutor(int corePoolSize,
               int maximumPoolSize,
               long keepAliveTime,
               TimeUnit unit,
               BlockingQueue<Runnable> workQueue,
               ThreadFactory threadFactory,
               RejectedExecutionHandler handler)
```

```java
public class BoundedExecutor {
    private final Executor exec;
    private final Semaphore semaphore;

    public BoundedExecutor(Executor exec, int bound) {
        this.exec = exec;
        this.semaphore = new Semaphore(bound);
    }

    public void submitTask(final Runnable command)
            throws InterruptedException, RejectedExecutionException {
        semaphore.acquire();
        try {
            exec.execute(new Runnable() {
                public void run() {
                    try {
                        command.run();
                    } finally {
                        semaphore.release();
                    }
                }
            });
        } catch (RejectedExecutionException e) {
            semaphore.release();
            throw e;
        }
    }
}
```

```java
RejectedExecutionHandler block = new RejectedExecutionHandler() {
  rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
     executor.getQueue().put( r );
  }
};

ThreadPoolExecutor pool = new ...
pool.setRejectedExecutionHandler(block);
```

# ExecutorService

并发 API 引入了 ExecutorService 作为一个在程序中直接使用 Thread 的高层次的替换方案。Executors 支持运行异步任务，通常管理一个线程池，这样一来我们就不需要手动去创建新的线程。在不断地处理任务的过程中，线程池内部线程将会得到复用，因此，在我们可以使用一个 executor service 来运行和我们想在我们整个程序中执行的一样多的并发任务。 下面是使用 executors 的第一个代码示例:

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

executor.submit(() -> {
    String threadName = Thread.currentThread().getName();
    System.out.println("Hello " + threadName);
});
// => Hello pool-1-thread-1
```

Executors 类提供了便利的工厂方法来创建不同类型的 executor services。在这个示例中我们使用了一个单线程线程池的 executor。 代码运行的结果类似于上一个示例，但是当运行代码时，你会注意到一个很大的差别：Java 进程从没有停止！Executors 必须显式的停止-否则它们将持续监听新的任务。 ExecutorService 提供了两个方法来达到这个目的——shutdwon()会等待正在执行的任务执行完而 shutdownNow()会终止所有正在执行的任务并立即关闭 execuotr。

```java
try {
    System.out.println("attempt to shutdown executor");
    executor.shutdown();
    executor.awaitTermination(5, TimeUnit.SECONDS);
    }
catch (InterruptedException e) {
    System.err.println("tasks interrupted");
}
finally {
    if (!executor.isTerminated()) {
        System.err.println("cancel non-finished tasks");
    }
    executor.shutdownNow();
    System.out.println("shutdown finished");
}
```

executor 通过等待指定的时间让当前执行的任务终止来“温柔的”关闭 executor。在等待最长 5 分钟的时间后，execuote 最终会通过中断所有的正在执行的任务关闭。一般来说 ExecutorService 与 Callable 经常协同使用，关于 Callable 的讲解可看下文的 Async 模块。

## 结合 Callable 与 Future 进行异步编程

Executors 支持通过 invokeAll()一次批量提交多个 callable。这个方法结果一个 callable 的集合，然后返回一个 future 的列表。

```java
ExecutorService executor = Executors.newWorkStealingPool();

List<Callable<String>> callables = Arrays.asList(
        () -> "task1",
        () -> "task2",
        () -> "task3");

executor.invokeAll(callables)
    .stream()
    .map(future -> {
        try {
            return future.get();
        }
        catch (Exception e) {
            throw new IllegalStateException(e);
        }
    })
    .forEach(System.out::println);
```

在这个例子中，我们利用 Java8 中的函数流(stream)来处理 invokeAll()调用返回的所有 future。我们首先将每一个 future 映射到它的返回值，然后将每个值打印到控制台。如果你还不属性 stream，可以阅读我的[Java8 Stream 教程](http://winterbe.com/posts/2014/07/31/java8-stream-tutorial-examples/)。

批量提交 callable 的另一种方式就是 invokeAny()，它的工作方式与 invokeAll()稍有不同。在等待 future 对象的过程中，这个方法将会阻塞直到第一个 callable 中止然后返回这一个 callable 的结果。 为了测试这种行为，我们利用这个帮助方法来模拟不同执行时间的 callable。这个方法返回一个 callable，这个 callable 休眠指定 的时间直到返回给定的结果。

```java
//这个callable方法是用来构造不同的Callable对象
Callable<String> callable(String result, long sleepSeconds) {
    return () -> {
        TimeUnit.SECONDS.sleep(sleepSeconds);
        return result;
    };
}
```

我们利用这个方法创建一组 callable，这些 callable 拥有不同的执行时间，从 1 分钟到 3 分钟。通过 invokeAny()将这些 callable 提交给一个 executor，返回最快的 callable 的字符串结果-在这个例子中为任务 2:

```java
ExecutorService executor = Executors.newWorkStealingPool();

List<Callable<String>> callables = Arrays.asList(
callable("task1", 2),
callable("task2", 1),
callable("task3", 3));

String result = executor.invokeAny(callables);
System.out.println(result);

// => task2
```

## 生命周期

xecutorService 接口继承了 Executor 接口，定义了一些生命周期的方法

```
public interface

ExecutorService extends Executor {
void shutdown();
List<Runnable> shutdownNow();
boolean isShutdown();
boolean isTerminated();
boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
}
```

1、shutdown 方法：这个方法会平滑地关闭 ExecutorService，当我们调用这个方法时， ExecutorService 停止接受任何新的任务且等待已经提交的任务执行完成(已经提交的任务会分两类：一类是已 经在执行的，另一类是还没有开始执行的)，当所有已经提交的任务执行完毕后将会关闭 ExecutorService。
2、awaitTermination 方法：这个方法有两个参数，一个是 timeout 即超 时时间，另一个是 unit 即时间单位。这个方法会使线程等待 timeout 时长，当超过 timeout 时间后，会监测 ExecutorService 是否已经关闭，若关闭则返回 true，否则返回 false。一般情况下会和 shutdown 方法组合使用 。

# ScheduledExecutorService: 定期执行

为了持续的多次执行常见的任务，我们可以利用调度线程池 ScheduledExecutorService 支持任务调度，持续执行或者延迟一段时间后执行。 下面的实例，调度一个任务在延迟 3 分钟后执行:

```java
ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

Runnable task = () -> System.out.println("Scheduling: " + System.nanoTime());
ScheduledFuture<?> future = executor.schedule(task, 3, TimeUnit.SECONDS);

TimeUnit.MILLISECONDS.sleep(1337);

long remainingDelay = future.getDelay(TimeUnit.MILLISECONDS);
System.out.printf("Remaining Delay: %sms", remainingDelay);
```

调度一个任务将会产生一个专门的 future 类型——ScheduleFuture，它除了提供了 Future 的所有方法之外，他还提供了 getDelay()方法来获得剩余的延迟。在延迟消逝后，任务将会并发执行。 为了调度任务持续的执行，executors 提供了两个方法 scheduleAtFixedRate()和 scheduleWithFixedDelay()。第一个方法用来以固定频率来执行一个任务，比如，下面这个示例中，每分钟一次:

```java
ScheduledExecutorService executor =     Executors.newScheduledThreadPool(1);

Runnable task = () -> System.out.println("Scheduling: " + System.nanoTime());

int initialDelay = 0;
int period = 1;
executor.scheduleAtFixedRate(task, initialDelay, period, TimeUnit.SECONDS);
```

另外，这个方法还接收一个初始化延迟，用来指定这个任务首次被执行等待的时长。 请记住：scheduleAtFixedRate()并不考虑任务的实际用时。所以，如果你指定了一个 period 为 1 分钟而任务需要执行 2 分钟，那么线程池为了性能会更快的执行。在这种情况下，你应该考虑使用 scheduleWithFixedDelay()。这个方法的工作方式与上我们上面描述的类似。不同之处在于等待时间 period 的应用是在一次任务的结束和下一个任务的开始之间。例如:

## ExecutorService 分类

```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;

public class Ch09_Executor {
   private static void run(ExecutorService threadPool) {
for(int i = 1; i < 5; i++) {
            final int taskID = i;
            threadPool.execute(new Runnable() {
                @Override
public void run() {
                    for(int i = 1; i < 5; i++) {
                        try {
                            Thread.sleep(20);// 为了测试出效果，让每次任务执行都需要一定时间
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        System.out.println("第" + taskID + "次任务的第" + i + "次执行");
                    }
                }
            });
        }
        threadPool.shutdown();// 任务执行完毕，关闭线程池
   }
    public static void main(String[] args) {
   // 创建可以容纳3个线程的线程池
   ExecutorService fixedThreadPool = Executors.newFixedThreadPool(3);
   // 线程池的大小会根据执行的任务数动态分配
   ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
            // 创建单个线程的线程池，如果当前线程在执行任务时突然中断，则会创建一个新的线程替代它继续执行任务
   ExecutorService singleThreadPool = Executors.newSingleThreadExecutor();
   // 效果类似于Timer定时器
   ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(3);

   run(fixedThreadPool);
//	   run(cachedThreadPool);
//	   run(singleThreadPool);
//	   run(scheduledThreadPool);
    }

}


```

### CachedThreadPool

CachedThreadPool 会创建一个缓存区，将初始化的线程缓存起来。会终止并且从缓存中移除已有 60 秒未被使用的线程。如果线程有可用的，就使用之前创建好的线程，如果线程没有可用的，就新创建线程。

- 重用：缓存型池子，先查看池中有没有以前建立的线程，如果有，就 reuse；如果没有，就建一个新的线程加入池中
- 使用场景：缓存型池子通常用于执行一些生存期很短的异步型任务，因此在一些面向连接的 daemon 型 SERVER 中用得不多。
- 超时：能 reuse 的线程，必须是 timeout IDLE 内的池中线程，缺省 timeout 是 60s，超过这个 IDLE 时长，线程实例将被终止及移出池。
- 结束：注意，放入 CachedThreadPool 的线程不必担心其结束，超过 TIMEOUT 不活动，其会自动被终止。

```
    // 线程池的大小会根据执行的任务数动态分配
    ExecutorService cachedThreadPool = Executors.newCachedThreadPool();

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0,                 //core pool size
                                      Integer.MAX_VALUE, //maximum pool size
                                      60L,               //keep alive time
                                      TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

执行结果，可以看出 4 个任务交替执行

```
第1次任务的第1次执行
第4次任务的第1次执行
第3次任务的第1次执行
第2次任务的第1次执行
第3次任务的第2次执行
第4次任务的第2次执行
第2次任务的第2次执行
第1次任务的第2次执行
第2次任务的第3次执行
第4次任务的第3次执行
第3次任务的第3次执行
第1次任务的第3次执行
第2次任务的第4次执行
第1次任务的第4次执行
第3次任务的第4次执行
第4次任务的第4次执行
```

### FixedThreadPool

在 FixedThreadPool 中，有一个固定大小的池。
如果当前需要执行的任务超过池大小，那么多出的任务处于等待状态，直到有空闲下来的线程执行任务，
如果当前需要执行的任务小于池大小，空闲的线程也不会去销毁。

- 重用：fixedThreadPool 与 cacheThreadPool 差不多，也是能 reuse 就用，但不能随时建新的线程
- 固定数目：其独特之处在于，任意时间点，最多只能有固定数目的活动线程存在，此时如果有新的线程要建立，只能放在另外的队列中等待，直到当前的线程中某个线程终止直接被移出池子
- 超时：和 cacheThreadPool 不同，FixedThreadPool 没有 IDLE 机制(可能也有，但既然文档没提，肯定非常长，类似依赖上层的 TCP 或 UDP IDLE 机制之类的)，
- 使用场景：所以 FixedThreadPool 多数针对一些很稳定很固定的正规并发线程，多用于服务器
- 源码分析：从方法的源代码看，cache 池和 fixed 池调用的是同一个底层池，只不过参数不同：
  fixed 池线程数固定，并且是 0 秒 IDLE(无 IDLE)
  cache 池线程数支持 0-Integer.MAX_VALUE(显然完全没考虑主机的资源承受能力)，60 秒 IDLE

```
// 创建可以容纳3个线程的线程池
ExecutorService fixedThreadPool = Executors.newFixedThreadPool(3);

public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, //core pool size
                                      nThreads, //maximum pool size
                                      0L,       //keep alive time
                                      TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}


```

执行结果：创建了一个固定大小的线程池，容量为 3，然后循环执行了 4 个任务。由输出结果可以看到，前 3 个任务首先执行完，然后空闲下来的线程去执行第 4 个任务。

```
第1次任务的第1次执行
第3次任务的第1次执行
第2次任务的第1次执行
第3次任务的第2次执行
第2次任务的第2次执行
第1次任务的第2次执行
第3次任务的第3次执行
第1次任务的第3次执行
第2次任务的第3次执行
第3次任务的第4次执行
第1次任务的第4次执行
第2次任务的第4次执行
第4次任务的第1次执行
第4次任务的第2次执行
第4次任务的第3次执行
第4次任务的第4次执行
```

### SingleThreadExecutor

SingleThreadExecutor 得到的是一个单个的线程，这个线程会保证你的任务执行完成。如果当前线程意外终止，会创建一个新线程继续执行任务，这和我们直接创建线程不同，也和 newFixedThreadPool(1)不同。

```
// 创建单个线程的线程池，如果当前线程在执行任务时突然中断，则会创建一个新的线程替代它继续执行任务
ExecutorService singleThreadPool = Executors.newSingleThreadExecutor();

public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1,  //core pool size
                                    1,  //maximum pool size
                                    0L, //keep alive time
                                    TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
}
```

执行结果：四个任务顺序执行

```
第1次任务的第1次执行
第1次任务的第2次执行
第1次任务的第3次执行
第1次任务的第4次执行
第2次任务的第1次执行
第2次任务的第2次执行
第2次任务的第3次执行
第2次任务的第4次执行
第3次任务的第1次执行
第3次任务的第2次执行
第3次任务的第3次执行
第3次任务的第4次执行
第4次任务的第1次执行
第4次任务的第2次执行
第4次任务的第3次执行
第4次任务的第4次执行
```

### ScheduledThreadPool

ScheduledThreadPool 是一个固定大小的线程池，与 FixedThreadPool 类似，执行的任务是定时执行。

```
// 效果类似于Timer定时器
ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(3);

public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize,      //core pool size
              Integer.MAX_VALUE, //maximum pool size
              0,                 //keep alive time
              TimeUnit.NANOSECONDS,
              new DelayedWorkQueue());
}
```

执行结果：它是定时执行的

```
第1次任务的第1次执行
第2次任务的第1次执行
第3次任务的第1次执行
第2次任务的第2次执行
第1次任务的第2次执行
第3次任务的第2次执行
第2次任务的第3次执行
第1次任务的第3次执行
第3次任务的第3次执行
第2次任务的第4次执行
第1次任务的第4次执行
第3次任务的第4次执行
第4次任务的第1次执行
第4次任务的第2次执行
第4次任务的第3次执行
第4次任务的第4次执行
```

## Concurrence Test

- [concurrency-torture-testing-your-code-within-the-java-memory-model](http://zeroturnaround.com/rebellabs/concurrency-torture-testing-your-code-within-the-java-memory-model/)

＃ ForkJoin:并行任务执行框架

- [聊聊并发(八)——Fork/Join 框架介绍](http://www.infoq.com/cn/articles/fork-join-introduction)
  Fork/Join 框架是 Java7 提供了的一个用于并行执行任务的框架， 是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。我们再通过 Fork 和 Join 这两个单词来理解下 Fork/Join 框架，Fork 就是把一个大任务切分为若干子任务并行的执行，Join 就是合并 这些子任务的执行结果，最后得到这个大任务的结果。比如计算 1+2+。。＋ 10000，可以分割成 10 个子任务，每个子任务分别对 1000 个数进行求和， 最终汇总这 10 个子任务的结果。Fork/Join 的运行流程图如下：
  ![](http://cdn3.infoqstatic.com/statics_s1_20160405-0343u1/resource/articles/fork-join-introduction/zh/resources/21.png)

## 工作窃取算法

工作窃取(work-stealing)算法是指某个线程从其他队列里窃取任务来执行。工作窃取的运行流程图如下：
![](http://cdn3.infoqstatic.com/statics_s1_20160405-0343u1/resource/articles/fork-join-introduction/zh/resources/image3.png)
那么为什么需要使用工作窃取算法呢？假如我们需要做一个比较大的任务，我们可以把这个任务分割为若干互不依赖的子任务，为了减少线程间的竞争，于是把这些 子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应，比如 A 线程负责处理 A 队列里的任务。但是有的线程 会先把自己队列里的任务干完，而其他线程对应的队列里还有任务等待处理。干完活的线程与其等着，不如去帮其他线程干活，于是它就去其他线程的队列里窃取一 个任务来执行。而在这时它们会访问同一个队列，所以为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列 的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。
工作窃取算法的优点是充分利用线程进行并行计算，并减少了线程间的竞争，其缺点是在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且消耗了更多的系统资源，比如创建多个线程和多个双端队列。

## 框架原理

我们已经很清楚 Fork/Join 框架的需求了，那么我们可以思考一下，如果让我们来设计一个 Fork/Join 框架，该如何设计？这个思考有助于你理解 Fork/Join 框架的设计。

第一步分割任务。首先我们需要有一个 fork 类来把大任务分割成子任务，有可能子任务还是很大，所以还需要不停的分割，直到分割出的子任务足够小。

第二步执行任务并合并结果。分割的子任务分别放在双端队列里，然后几个启动线程分别从双端队列里获取任务执行。子任务执行完的结果都统一放在一个队列里，启动一个线程从队列里拿数据，然后合并这些数据。

Fork/Join 使用两个类来完成以上两件事情：

- ForkJoinTask：我们要使用 ForkJoin 框架，必须首先创建一个 ForkJoin 任务。它提供在任务中执行 fork()和 join()操作的机制，通常情况下我们不需要直接继承 ForkJoinTask 类，而只需要继承它的子类，Fork/Join 框架提供了以下两个子类:
  - RecursiveAction：用于没有返回结果的任务。
  - RecursiveTask ：用于有返回结果的任务。
- ForkJoinPool ：ForkJoinTask 需要通过 ForkJoinPool 来执行，任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务。

## Usage

让我们通过一个简单的需求来使用下 Fork／Join 框架，需求是：计算 1+2+3+4 的结果。
使用 Fork／Join 框架首先要考虑到的是如何分割任务，如果我们希望每个子任务最多执行两个数的相加，那么我们设置分割的阈值是 2，由于是 4 个 数字相加，所以 Fork／Join 框架会把这个任务 fork 成两个子任务，子任务一负责计算 1+2，子任务二负责计算 3+4，然后再 join 两个子任务的 结果。
因为是有结果的任务，所以必须继承 RecursiveTask，实现代码如下：
完整代码查看[CountTask](https://github.com/wxyyxc1992/WXJavaToolkits/blob/master/src/main/java/wx/toolkits/sysproc/concurrence/forkjoin/CountTask.java)

```
            CountTask leftTask = new CountTask(start, middle);
            CountTask rightTask = new CountTask(middle + 1, end);
            //执行子任务
            leftTask.fork();
            rightTask.fork();
            //等待子任务执行完，并得到其结果
            int leftResult = (int) leftTask.join();
            int rightResult = (int) rightTask.join();
            //合并子任务
            sum = leftResult + rightResult;
```

通过这个例子让我们再来进一步了解 ForkJoinTask，ForkJoinTask 与一般的任务的主要区别在于它需要实现 compute 方法，在这个 方法里，首先需要判断任务是否足够小，如果足够小就直接执行任务。如果不足够小，就必须分割成两个子任务，每个子任务在调用 fork 方法时，又会进入 compute 方法，看看当前子任务是否需要继续分割成孙任务，如果不需要继续分割，则执行当前子任务并返回结果。使用 join 方法会等待子任务执行完并 得到其结果。

### 异常处理

ForkJoinTask 在执行的时候可能会抛出异常，但是我们没办法在主线程里直接捕获异常，所以 ForkJoinTask 提供了 isCompletedAbnormally()方法来检查任务是否已经抛出异常或已经被取消了，并且可以通过 ForkJoinTask 的 getException 方法获取异常。使用如下代码：

```
if(task.isCompletedAbnormally())
{
    System.out.println(task.getException());
}
```

getException 方法返回 Throwable 对象，如果任务被取消了则返回 CancellationException。如果任务没有完成或者没有抛出异常则返回 null。

## 实现原理

ForkJoinPool 由 ForkJoinTask 数组和 ForkJoinWorkerThread 数组组成，ForkJoinTask 数组负责存放程序提交给 ForkJoinPool 的任务，而 ForkJoinWorkerThread 数组负责执行这些任务。
ForkJoinTask 的 fork 方法实现原理。当我们调用 ForkJoinTask 的 fork 方法时，程序会调用 ForkJoinWorkerThread 的 pushTask 方法异步的执行这个任务，然后立即返回结果。代码如下：

```
public final ForkJoinTask fork() {
    ((ForkJoinWorkerThread) Thread.currentThread()).pushTask(this);
    return this;
}
```

pushTask 方法把当前任务存放在 ForkJoinTask 数组 queue 里。然后再调用 ForkJoinPool 的 signalWork()方法唤醒或创建一个工作线程来执行任务。代码如下：

```
final void pushTask(ForkJoinTask t) {
        ForkJoinTask[] q; int s, m;
        if ((q = queue) != null) {    // ignore if queue removed
            long u = (((s = queueTop) & (m = q.length - 1)) << ASHIFT) + ABASE;
            UNSAFE.putOrderedObject(q, u, t);
            queueTop = s + 1;         // or use putOrderedInt
            if ((s -= queueBase) <= 2)
                pool.signalWork();
	else if (s == m)
                growQueue();
        }
    }
```

ForkJoinTask 的 join 方法实现原理。Join 方法的主要作用是阻塞当前线程并等待获取结果。让我们一起看看 ForkJoinTask 的 join 方法的实现，代码如下：

```
public final V join() {
        if (doJoin() != NORMAL)
            return reportResult();
        else
            return getRawResult();
}
private V reportResult() {
        int s; Throwable ex;
        if ((s = status) == CANCELLED)
            throw new CancellationException();
if (s == EXCEPTIONAL && (ex = getThrowableException()) != null)
            UNSAFE.throwException(ex);
        return getRawResult();
}
```

首先，它调用了 doJoin()方法，通过 doJoin()方法得到当前任务的状态来判断返回什么结果，任务状态有四种：已完成(NORMAL)，被取消(CANCELLED)，信号(SIGNAL)和出现异常(EXCEPTIONAL)。

- 如果任务状态是已完成，则直接返回任务结果。
- 如果任务状态是被取消，则直接抛出 CancellationException。
- 如果任务状态是抛出异常，则直接抛出对应的异常。

让我们再来分析下 doJoin()方法的实现代码：

```java
private int doJoin() {
        Thread t; ForkJoinWorkerThread w; int s; boolean completed;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) {
            if ((s = status) < 0)
 return s;
            if ((w = (ForkJoinWorkerThread)t).unpushTask(this)) {
                try {
                    completed = exec();
                } catch (Throwable rex) {
                    return setExceptionalCompletion(rex);
                }
                if (completed)
                    return setCompletion(NORMAL);
            }
            return w.joinTask(this);
        }
        else
            return externalAwaitDone();
    }

```

在 doJoin()方法里，首先通过查看任务的状态，看任务是否已经执行完了，如果执行完了，则直接返回任务状态，如果没有执行完，则从任务数组里取出任务并执行。如果任务顺利执行完成了，则设置任务状态为 NORMAL，如果出现异常，则纪录异常，并将任务状态设置为 EXCEPTIONAL。

# Thread Communication(线程通信)

## Exchanger

```
import java.util.concurrent.Exchanger;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;


public class ExchangeTest {
public static void main(String[] args) {
ExecutorService service =Executors.newCachedThreadPool();
final Exchanger exchanger = new Exchanger();
service.execute(new Runnable() {
@Override
public void run() {
try{
String data1 = "零食";
System.out.println("线程"+Thread.currentThread().getName()+
"正在把数据 "+data1+" 换出去");
Thread.sleep((long)Math.random()*10000);
String data2 = (String)exchanger.exchange(data1);
System.out.println("线程 "+Thread.currentThread().getName()+
"换回的数据为 "+data2);
}catch(Exception e){
e.printStackTrace();
}
}
});
service.execute(new Runnable() {
@Override
public void run() {
try{
String data1 = "钱";
System.out.println("线程"+Thread.currentThread().getName()+
"正在把数据 "+data1+" 交换出去");
Thread.sleep((long)(Math.random()*10000));
String data2 =(String)exchanger.exchange(data1);
System.out.println("线程 "+Thread.currentThread().getName()+
"交换回来的数据是: "+data2);
}catch(Exception e){
e.printStackTrace();
}
}
});
}
}
```

最后结果为：

```
线程pool-1-thread-1正在把数据 零食 换出去
线程pool-1-thread-2正在把数据 钱 交换出去
线程 pool-1-thread-1换回的数据为 钱
线程 pool-1-thread-2交换回来的数据是: 零食
```

# Asynchronous(异步)

### Timeouts

任何 future.get()调用都会阻塞，然后等待直到 callable 中止。在最糟糕的情况下，一个 callable 持续运行——因此使你的程序将没有响应。我们可以简单的传入一个时长来避免这种情况。

```java
ExecutorService executor = Executors.newFixedThreadPool(1);

Future<Integer> future = executor.submit(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
        return 123;
    }
    catch (InterruptedException e) {
        throw new IllegalStateException("task interrupted", e);
    }
});

future.get(1, TimeUnit.SECONDS);
```

运行上面的代码将会产生一个 TimeoutException:

```java
Exception in thread "main" java.util.concurrent.TimeoutException
    at java.util.concurrent.FutureTask.get(FutureTask.java:205)
```

## Promise
