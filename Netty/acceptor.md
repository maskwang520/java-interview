### Reactor 完全揭秘
![Netty线程模型](threadmodel.png)
Reactor对应的实现为NioEventLoop，NioEventLoop中维护了一个线程，线程启动时会调用NioEventLoop的run方法，执行I/O任务和非I/O任务。
* I/O任务即selectionKey中ready的事件，如accept、connect、read、write等，由processSelectedKeysOptimized或processSelectedKeysPlain方法触发。
* 非IO任务则为添加到taskQueue中的任务，如register0、bind0等任务，由runAllTasks方法触发。
* 两种任务的执行时间比由变量ioRatio控制，默认为50，则表示允许非IO任务执行的时间与IO任务的执行时间相等。
`SingleThreadEventExecutor`
```java
public void execute(Runnable task) {
        if(task == null) {
            throw new NullPointerException("task");
        } else {
            // netty会判断reactor线程有没有被启动，如果没有被启动，那就启动线程再往任务队列里面添加任务
            boolean inEventLoop = this.inEventLoop();
            if(inEventLoop) {
                this.addTask(task);
            } else {
                //FastThreadLocalThread 执行NioEventLoop的run方法，启动Reactor线程
                this.startThread();
                this.addTask(task);
                if(this.isShutdown() && this.removeTask(task)) {
                    reject();
                }
            }

            if(!this.addTaskWakesUp && this.wakesUpForTask(task)) {
                this.wakeup(inEventLoop);
            }

        }
    }
```
* 启动Reactor线程
接着执行NioEventLoo的run方法
```java
protected void run() {
    //死循环，不断的运行
        while(true) {
            //每次循环把wakenUp初始化为false
            boolean oldWakenUp = this.wakenUp.getAndSet(false);

            try {
                //有任务，则立刻返回
                if(this.hasTasks()) {
                    this.selectNow();
                } else {
                    //执行下面的select，会阻塞
                    this.select(oldWakenUp);
                    if(this.wakenUp.get()) {
                        //唤醒
                        this.selector.wakeup();
                    }
                }

                this.cancelledKeys = 0;
                this.needsToSelectAgain = false;
                int ioRatio = this.ioRatio;
                //处理Io和加入的任务是各占50%
                if(ioRatio == 100) {
                    //处理Io事件
                    this.processSelectedKeys();
                    //运行加入的任务（非IO）
                    this.runAllTasks();
                } else {
                    long ioStartTime = System.nanoTime();
                    this.processSelectedKeys();
                    long ioTime = System.nanoTime() - ioStartTime;
                    this.runAllTasks(ioTime * (long)(100 - ioRatio) / (long)ioRatio);
                }
                //关闭
                if(this.isShuttingDown()) {
                    this.closeAll();
                    if(this.confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable var8) {
                logger.warn("Unexpected exception in the selector loop.", var8);

                try {
                    Thread.sleep(1000L);
                } catch (InterruptedException var7) {
                    ;
                }
            }
        }
    }
```
执行select操作
```java
private void select(boolean oldWakenUp) throws IOException {
        Selector selector = this.selector;

        try {
            int selectCnt = 0;
            //记录select阻塞之前的时间
            long currentTimeNanos = System.nanoTime();
            long selectDeadLineNanos = currentTimeNanos + this.delayNanos(currentTimeNanos);

            while(true) {
                //设置延迟时间
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
                if(timeoutMillis <= 0L) {
                    if(selectCnt == 0) {
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    break;
                }
                //阻塞在这里
                int selectedKeys = selector.select(timeoutMillis);
                ++selectCnt;
                //如果有Io事件，或者有任务，或者用户唤醒的，或者定时任务快到了
                if(selectedKeys != 0 || oldWakenUp || this.wakenUp.get() || this.hasTasks() || this.hasScheduledTasks()) {
                    break;
                }

                if(Thread.interrupted()) {
                    if(logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely because Thread.currentThread().interrupt() was called. Use NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                    }
                    selectCnt = 1;
                    break;
                }
                //记录系统当前时间，如果当前时间-select阻塞之前时间>timeoutMillis,则说明是正常的唤醒，否则有可能是jdk的Java epoll空轮询bug
                long time = System.nanoTime();
                if(time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                    //重置1
                    selectCnt = 1;

                } else if(SELECTOR_AUTO_REBUILD_THRESHOLD > 0 && selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                    logger.warn("Selector.select() returned prematurely {} times in a row; rebuilding selector.", Integer.valueOf(selectCnt));
                    //Java epoll空轮询bug,则构建一个新的selector。默认selectCnt=500就认为出现了这个bug
                    this.rebuildSelector();
                    selector = this.selector;
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                currentTimeNanos = time;
            }

            if(selectCnt > 3 && logger.isDebugEnabled()) {
                logger.debug("Selector.select() returned prematurely {} times in a row.", Integer.valueOf(selectCnt - 1));
            }
        } catch (CancelledKeyException var13) {
            if(logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector - JDK bug?", var13);
            }
        }

    }
```
* rebuildSelector（）把旧的channel注册到新的Selector上，取消在旧的selector上的注册key。
步骤：
1. 如果有轮询到IO事件，oldWakenUp 参数为true，任务队列里面有任务（hasTasks）
第一个定时任务即将要被执行 （hasScheduledTasks（）），用户主动唤醒（wakenUp.get()）则selct返回，否则继续阻塞
2. 如何处理Java epoll空轮询bug。（重构Selector)。

Reactor的任务如下：
![Reactor](https://upload-images.jianshu.io/upload_images/1357217-67ed6d1e8070426f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

#### processSelectedKeys()
根据选中的key，作相应的操作
```java
private void processSelectedKeysOptimized(SelectionKey[] selectedKeys) {
        int i = 0;
        //死循环，直到处理完
        while(true) {
            //selct 的key数组
            SelectionKey k = selectedKeys[i];
            //说明处理完了
            if(k == null) {
                return;
            }
            //有利于Gc,防止内存泄漏
            selectedKeys[i] = null;
            Object a = k.attachment();
            if(a instanceof AbstractNioChannel) {
                //如果是AbstractNioChannel
                processSelectedKey(k, (AbstractNioChannel)a);
            } else {
                //如果是NioTask
                NioTask<SelectableChannel> task = (NioTask)a;
                processSelectedKey(k, task);
            }
            //是否需要再次Select
            if(this.needsToSelectAgain) {
                //清理
                while(selectedKeys[i] != null) {
                    selectedKeys[i] = null;
                    ++i;
                }
                //重新选一遍
                this.selectAgain();
                selectedKeys = this.selectedKeys.flip();
                i = -1;
            }

            ++i;
        }
    }
```
* 按理说Selected的是Set结合，Netty在这里是做了优化，用数组来表示。这样的查找能O(1)
* 在把channel注册到Selector上的时候，其实将Jdk底层channel注册到Selector上，并把AbstractChannel作为attachMent。这样的话可以select中的channel取出AbstractChannel。
* needsToSelectAgain是为了256次channel断线，重新清理一下selectionKey，保证现存的SelectionKey及时有效。
```java
    //根据选择的key代表的事件，执行相应的操作
    private static void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        NioUnsafe unsafe = ch.unsafe();
        if(!k.isValid()) {
            unsafe.close(unsafe.voidPromise());
        } else {
            try {
                int readyOps = k.readyOps();
                if((readyOps & 17) != 0 || readyOps == 0) {
                    //如果是读事件是读事件
                    unsafe.read();
                    if(!ch.isOpen()) {
                        return;
                    }
                }

                if((readyOps & 4) != 0) {
                    //如果是写事件，是强制write事件
                    ch.unsafe().forceFlush();
                }
                //如果是Connect的事件，则修改为Accept，并完成连接
                if((readyOps & 8) != 0) {
                    int ops = k.interestOps();
                    ops &= -9;
                    k.interestOps(ops);
                    unsafe.finishConnect();
                }
            } catch (CancelledKeyException var5) {
                unsafe.close(unsafe.voidPromise());
            }

        }
    }
```

processSelectedKeys主要是根据SelectionKey来做相应的read,write,connect等操作。这些过程中，Netty做了些优化。

#### runAllTasks
这是第三步，处理非IO的任务.主要有以下3中场景
1. 用户自定义普通任务
```java
ctx.channel().eventLoop().execute(new Runnable() {
    @Override
    public void run() {
        //...
    }
});

//SingleThreadEventExecutor中
public void execute(Runnable task) {
        if(task == null) {
            throw new NullPointerException("task");
        } else {
            boolean inEventLoop = this.inEventLoop();
            //不管怎样都保证在Reactor里面处理
            if(inEventLoop) {
                this.addTask(task);
            } else {
                this.startThread();
                this.addTask(task);
                if(this.isShutdown() && this.removeTask(task)) {
                    reject();
                }
            }

            if(!this.addTaskWakesUp && this.wakesUpForTask(task)) {
                this.wakeup(inEventLoop);
            }

        }
    }
//添加任务
protected void addTask(Runnable task) {
        if(task == null) {
            throw new NullPointerException("task");
        } else {
            if(this.isShutdown()) {
                reject();
            }
            //用LinkedBlockingQueue来保存
            this.taskQueue.add(task);
        }
    }
```

2. 非当前reactor线程调用channel的各种方法
```java
channel.write(...)
```
最终会调用
```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
        AbstractChannelHandlerContext next = this.findContextOutbound();
        EventExecutor executor = next.executor();
        if(executor.inEventLoop()) {
            next.invokeWrite(msg, promise);
            if(flush) {
                next.invokeFlush();
            }
        } else {
            Object task;
            if(flush) {
                task = AbstractChannelHandlerContext.WriteAndFlushTask.newInstance(next, msg, promise);
            } else {
                task = AbstractChannelHandlerContext.WriteTask.newInstance(next, msg, promise);
            }

            safeExecute(executor, (Runnable)task, promise, msg);
        }

    }
```
走else分支，就是非当前Reactor线程。剩下也是走第一种一样的，只是所属线程有区别。

3. 用户自定义定时任务
```java
 <V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
        if(this.inEventLoop()) {
            //加入定时任务队列里面
            this.scheduledTaskQueue().add(task);
        } else {
            //非Reactor线程执行，类似于第二种自定义任务
            this.execute(new OneTimeTask() {
                public void run() {
                    AbstractScheduledEventExecutor.this.scheduledTaskQueue().add(task);
                }
            });
        }

        return task;
    }
```

```java
public int compareTo(Delayed o) {
        if(this == o) {
            return 0;
        } else {
            ScheduledFutureTask<?> that = (ScheduledFutureTask)o;
            long d = this.deadlineNanos() - that.deadlineNanos();
            //根据执行时间的长短排序
            if(d < 0L) {
                return -1;
            } else if(d > 0L) {
                return 1;
            } else if(this.id < that.id) {
                return -1;
            } else if(this.id == that.id) {
                throw new Error();
            } else {
                return 1;
            }
        }
    }
 //   定时任务类型
//1.若干时间后执行一次
//2.每隔一段时间执行一次
//3.每次执行结束，隔一定时间再执行一次
 public void run() {
        assert this.executor().inEventLoop();

        try {
            //若干时间后执行一次
            if(this.periodNanos == 0L) {
                if(this.setUncancellableInternal()) {
                    V result = this.task.call();
                    //这种类型执行一次就结束
                    this.setSuccessInternal(result);
                }
            } else if(!this.isCancelled()) {
                //先执行任务
                this.task.call();
                if(!this.executor().isShutdown()) {
                    long p = this.periodNanos;
                    //表示是以固定频率执行某个任务
                    if(p > 0L) {
                        this.deadlineNanos += p;
                    } else {
                        //每次任务执行完毕之后，间隔多长时间之后再次执行
                        this.deadlineNanos = nanoTime() - p;
                    }

                    if(!this.isCancelled()) {
                        Queue<ScheduledFutureTask<?>> scheduledTaskQueue = ((AbstractScheduledEventExecutor)this.executor()).scheduledTaskQueue;

                        assert scheduledTaskQueue != null;
                        //又加入进去
                        scheduledTaskQueue.add(this);
                    }
                }
            }
        } catch (Throwable var4) {
            this.setFailureInternal(var4);
        }

    }
```

```java
protected boolean runAllTasks(long timeoutNanos) {
        this.fetchFromScheduledTaskQueue();
        //取出第一个任务
        Runnable task = this.pollTask();
        if(task == null) {
            return false;
        } else {
            //保证任务不能超过deadline
            long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
            long runTasks = 0L;

            long lastExecutionTime;
            while(true) {
                try {
                    //正式的执行
                    task.run();
                } catch (Throwable var11) {
                    logger.warn("A task raised an exception.", var11);
                }
                //处理的任务数+1
                ++runTasks;
                //如果处理的任务已经达到64个
                if((runTasks & 63L) == 0L) {

                    lastExecutionTime = ScheduledFutureTask.nanoTime();
                     //判断当前时间是否超过reactor任务循环的截止时间
                    if(lastExecutionTime >= deadline) {
                        break;
                    }
                }
                //又取出任务
                task = this.pollTask();
                if(task == null) {

                    lastExecutionTime = ScheduledFutureTask.nanoTime();
                    break;
                }
            }
            //简单记录一下任务执行的时间
            this.lastExecutionTime = lastExecutionTime;
            return true;
        }
    }

    private void fetchFromScheduledTaskQueue() {
        if(this.hasScheduledTasks()) {
            long nanoTime = AbstractScheduledEventExecutor.nanoTime();

            while(true) {
                //去Priority队列里面取第一个Task
                Runnable scheduledTask = this.pollScheduledTask(nanoTime);
                //直到取到的任务为null,则跳出循环
                if(scheduledTask == null) {
                    break;
                }
                //把快要到执行时间的任务加入到LinkedBlockingQueue
                this.taskQueue.add(scheduledTask);
            }
        }

    }
```
RunTask总结：
* 当前reactor线程调用当前eventLoop执行任务，直接执行，否则，添加到任务队列稍后执行
* netty内部的任务分为普通任务和定时任务，分别落地到LinkedBlokingQueue和PriorityQueue
* netty每次执行任务循环之前，会将已经到期的定时任务从PriorityQueue转移到LinkedBlokingQueue
* netty每隔64个任务检查一下是否该退出任务循环


参考文章：
https://www.jianshu.com/p/58fad8e42379  




