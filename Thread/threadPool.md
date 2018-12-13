#### ThreadPool

1. 创建线程池
```java
/* @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @param threadFactory the factory to use when the executor
     *        creates a new thread
     * @param handler the handler to use when execution is blocked
     *        because the thread bounds and queue capacities are reached
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
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
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
* 超过corePoolSize会在keepAliveTime之后死亡
2. 线程池的5种状态
```java
// runState is stored in the high-order bits
    //Accept new tasks and process queued tasks
    private static final int RUNNING    = -1 << COUNT_BITS; 
    // Don't accept new tasks, but process queued tasks
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    //Don't accept new tasks, don't process queued tasks,and interrupt in-progress tasks
    private static final int STOP       =  1 << COUNT_BITS;
    //All tasks have terminated, workerCount is zero,the thread transitioning to state TIDYING will run the terminated() hook method
    private static final int TIDYING    =  2 << COUNT_BITS;
    //terminated() has completed
    private static final int TERMINATED =  3 << COUNT_BITS;
```
3. 四种拒绝策略

* AbortPolicy（丢弃并抛出异常）
```java
public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```
* DiscardPolicy（直接丢弃，不做任何事）
```java
 public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        /**
         * Does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
```
* DiscardOldestPolicy（丢弃最旧的任务）
```java
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```
* CallerRunsPolicy(在用户线程里面处理)
```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code CallerRunsPolicy}.
         */
        public CallerRunsPolicy() { }

        /**
         * Executes task r in the caller's thread, unless the executor
         * has been shut down, in which case the task is discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```
4. 线程的状态

* 新建(New)
```java
Thread thread = new Thread();
```

* Runnable
线程对象创建后，其他线程(比如main线程）调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中 获取cpu 的使用权 。

* Running
可运行状态(runnable)的线程获得了cpu 时间片（timeslice） ，执行程序代码。

* 限期等待（Timed Waiting）
也可以称作 TIMED_WAITING（有等待时间的等待状态）。
线程主动调用以下方法：
Thread.sleep方法；
Object的wait方法，带有时间；
Thread.join方法，带有时间；
LockSupport的parkNanos方法，带有时间。

* 无限期等待（Waiting）
运行中（Running）的线程执行了以下方法：
Object的wait方法，并且没有使用timeout参数;
Thread的join方法，没有使用timeout参数；
LockSupport的park方法；
Conditon的await方法。

* 阻塞
阻塞状态是指线程因为某种原因放弃了cpu 使用权，暂时停止运行。直到线程进入可运行(runnable)状态，才有机会再次获得cpu timeslice 转到运行(running)状态。阻塞的情况分两种：

同步阻塞：运行(running)的线程进入了一个synchronized方法，若该同步锁被别的线程占用，则JVM会把该线程放入锁池(lock pool)中。
其他阻塞：运行(running)的线程发出了I/O请求时，JVM会把该线程置为阻塞状态。当I/O处理完毕时，线程重新转入可运行(runnable)状态。

* 终止
线程run()、main() 方法执行结束，或者因异常退出了run()方法，则该线程结束生命周期。

4. 四种线程池
```java
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
```
* KeepAliveTime为0，队列是无限的，最大线程数=core

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
* 与上面的一样，只不过，core=max=1;

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
* 用的SynchronousQueue，只能保存一个任务，后来的任务需要阻塞。

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```
* scheduledThreadPool.schedule(worker, 5, TimeUnit.SECONDS);(只执行一次)
// 周期性执行，每5秒执行一次
scheduledThreadPool.scheduleAtFixedRate(worker, 0,5, TimeUnit.SECONDS);


