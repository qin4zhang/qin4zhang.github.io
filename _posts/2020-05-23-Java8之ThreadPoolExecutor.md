---
layout:     post
title:      Java8之ThreadPoolExecutor
subtitle:   线程池
date:       2020-05-23
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java
    - ThreadPoolExecutor

---
# 注意
> 想法及时记录，实现可以待做。

## 简介
线程池解决了两个不同的问题：由于减少了每个任务的调用开销，它们通常在执行大量异步任务时提供更好的性能。

并且它们提供了一种约束和管理资源的方法，包括线程在执行任务集合时消耗的资源。每个任务还维护一些基本的统计数据，如已完成的任务数量。

ThreadPoolExecutor的UML类图如图所示：

![ThreadPoolExecutor]({{site.url}}/img/java/ThreadPoolExecutor.png)

### 组成介绍

```
    //  状态控制的变量，32位的原子变量
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    // 数量的位数，32-3=29位，也即最大就是 2^29 -1 个
    private static final int COUNT_BITS = Integer.SIZE - 3;
    // 最大数量 2^29 -1
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // 线程池运行状态，是由int的高位存储的，高3位表示。通过移位运算得到值的大小。
    // 整个状态值的大小顺序主: RUNNING < SHUTDOWN < STOP < TIDYING < TERMINATED
    // runState is stored in the high-order bits
    // 高位 111 ... -1的二进制表示，32个1，带符号位左移29位后，就剩下高位的3个1。该状态表示线程池会接受新的任务，并处理阻塞队列的任务。
    private static final int RUNNING    = -1 << COUNT_BITS; 
    // 高位 000 ... 该状态表示 不会接受新的任务，但是还会继续处理阻塞队列中的任务
    private static final int SHUTDOWN   =  0 << COUNT_BITS; 
    // 高位 001 ... 该状态表示 不会接受新的任务，也不会处理组赛队列的任务，而且还会中断正在运行中的任务
    private static final int STOP       =  1 << COUNT_BITS; 
    // 高位 010 ... 该状态表示 所有任务都已经终止
    private static final int TIDYING    =  2 << COUNT_BITS; 
    // 高位 011 ... 改装套表示 terminated方法执行完成
    private static final int TERMINATED =  3 << COUNT_BITS; 

    // Packing and unpacking ctl
    // 状态值取高3位
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // 数量取后29位
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    // 位 或运算，将状态值和数量组成 ctl状态控制量
    private static int ctlOf(int rs, int wc) { return rs | wc; }


    // 工作任务队列
    private final BlockingQueue<Runnable> workQueue;
    // 工作线程集合
    private final HashSet<Worker> workers = new HashSet<Worker>();
    // 完成的任务数
    private long completedTaskCount;
    // 线程工厂
    private volatile ThreadFactory threadFactory;
    // 拒绝策略
    private volatile RejectedExecutionHandler handler;

    /**
     * 空闲线程存活时间
     * Timeout in nanoseconds for idle threads waiting for work.
     * Threads use this timeout when there are more than corePoolSize
     * present or if allowCoreThreadTimeOut. Otherwise they wait
     * forever for new work.
     */
    private volatile long keepAliveTime;


    /**
     * 核心线程数
     * Core pool size is the minimum number of workers to keep alive
     * (and not allow to time out etc) unless allowCoreThreadTimeOut
     * is set, in which case the minimum is zero.
     */
    private volatile int corePoolSize;

    /**
     * 最大线程数
     * Maximum pool size. Note that the actual maximum is internally
     * bounded by CAPACITY.
     */
    private volatile int maximumPoolSize;

```

### 关键内部类

```

    /**
     * Class Worker mainly maintains interrupt control state for
     * threads running tasks, along with other minor bookkeeping.
     * This class opportunistically extends AbstractQueuedSynchronizer
     * to simplify acquiring and releasing a lock surrounding each
     * task execution.  This protects against interrupts that are
     * intended to wake up a worker thread waiting for a task from
     * instead interrupting a task being run.  We implement a simple
     * non-reentrant mutual exclusion lock rather than use
     * ReentrantLock because we do not want worker tasks to be able to
     * reacquire the lock when they invoke pool control methods like
     * setCorePoolSize.  Additionally, to suppress interrupts until
     * the thread actually starts running tasks, we initialize lock
     * state to a negative value, and clear it upon start (in
     * runWorker).
     */
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        // worker持有的线程，只有线程工厂运行失败的时候，才会为null
        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        // 初始化任务，不为空的时候，任务直接运行，不需要添加到队列中
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        // 完成的任务数
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * 构造工作worker
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
        
        /** Delegates main run loop to outer runWorker  */
        public void run() {
            // 当这个worker执行的时候，这个方法便会执行，里面是一直在循环取任务执行任务
            runWorker(this);
        }
    }
```

### 提交任务

```

    /**
     * 执行任务，要么是新的线程，要么是池化的线程
     * 如果当前线程数小于核心线程，那么执行任务就会直接新开线程执行
     * 如果任务加入队列成功，就检查是否要新开线程
     * 如果任务加入队列失败，就有可能是线程池关闭或者队列满了，所以执行拒绝策略
     * Executes the given task sometime in the future.  The task
     * may execute in a new thread or in an existing pooled thread.
     *
     * If the task cannot be submitted for execution, either because this
     * executor has been shutdown or because its capacity has been reached,
     * the task is handled by the current {@code RejectedExecutionHandler}.
     *
     * @param command the task to execute
     * @throws RejectedExecutionException at discretion of
     *         {@code RejectedExecutionHandler}, if the task
     *         cannot be accepted for execution
     * @throws NullPointerException if {@code command} is null
     */
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        // 获取状态控制量，可以得到运行状态和线程数
        int c = ctl.get();
        // 如果当前线程数小于核心数
        if (workerCountOf(c) < corePoolSize) {
            // 增加核心工作线程成功，本次执行完成
            if (addWorker(command, true))
                return;
            // 增加失败，再次获取状态控制量
            c = ctl.get();
        }
        // 核心线程数满了，那么如果线程池的状态在「运行中」并且加入到队列中
        if (isRunning(c) && workQueue.offer(command)) {
            // 再次检查状态量，如果线程池状态不在「运行中」，并且删除任务成功，则执行拒绝策略
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果线程数为0，则增加非核心工作线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 队列也满了，那么就尝试增加非核心工作线程，如果失败，则拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }

```

### 添加任务

```
    /**
     * Checks if a new worker can be added with respect to current
     * pool state and the given bound (either core or maximum). If so,
     * the worker count is adjusted accordingly, and, if possible, a
     * new worker is created and started, running firstTask as its
     * first task. This method returns false if the pool is stopped or
     * eligible to shut down. It also returns false if the thread
     * factory fails to create a thread when asked.  If the thread
     * creation fails, either due to the thread factory returning
     * null, or due to an exception (typically OutOfMemoryError in
     * Thread.start()), we roll back cleanly.
     *
     * @param firstTask the task the new thread should run first (or
     * null if none). Workers are created with an initial first task
     * (in method execute()) to bypass queuing when there are fewer
     * than corePoolSize threads (in which case we always start one),
     * or when the queue is full (in which case we must bypass queue).
     * Initially idle threads are usually created via
     * prestartCoreThread or to replace other dying workers.
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 已经shutdown, firstTask 为空的添加并不会成功
            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                // 工作线程数超过最大值，或者超过设置的值，都不能再次新增，返回失败
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // cas自旋增加工作线程数，成功就跳出本次循环，进入下面的逻辑
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 重读状态量，前后不一致说明有变化，需要重新循环一次
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 构造新的worker
            w = new Worker(firstTask);
            final Thread t = w.thread;
            // worker的绑定线程不为null,只有线程工厂创建失败的时候，才会为null
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    // 线程池状态为「运行中」或者 「SHUTDOWN」但还是会处理剩下的待处理任务，也就是不会是初始化的firstTask，即firstTask为null
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        // 如果worker的绑定线程已经启动了，则状态异常
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 添加进线程集合中
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 已经将worker添加进线程集合中，那么可以启动线程执行任务了
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            // worker如果没有启动成功，则添加失败，清除相关的数据
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

```

### 执行任务

这个是主要的执行队列任务的方法，他会一直循环获取队列里面的worker，然后执行。

```

    /**
     * Main worker run loop.  Repeatedly gets tasks from queue and
     * executes them, while coping with a number of issues:
     *
     * 1. We may start out with an initial task, in which case we
     * don't need to get the first one. Otherwise, as long as pool is
     * running, we get tasks from getTask. If it returns null then the
     * worker exits due to changed pool state or configuration
     * parameters.  Other exits result from exception throws in
     * external code, in which case completedAbruptly holds, which
     * usually leads processWorkerExit to replace this thread.
     *
     * 2. Before running any task, the lock is acquired to prevent
     * other pool interrupts while the task is executing, and then we
     * ensure that unless pool is stopping, this thread does not have
     * its interrupt set.
     *
     * 3. Each task run is preceded by a call to beforeExecute, which
     * might throw an exception, in which case we cause thread to die
     * (breaking loop with completedAbruptly true) without processing
     * the task.
     *
     * 4. Assuming beforeExecute completes normally, we run the task,
     * gathering any of its thrown exceptions to send to afterExecute.
     * We separately handle RuntimeException, Error (both of which the
     * specs guarantee that we trap) and arbitrary Throwables.
     * Because we cannot rethrow Throwables within Runnable.run, we
     * wrap them within Errors on the way out (to the thread's
     * UncaughtExceptionHandler).  Any thrown exception also
     * conservatively causes thread to die.
     *
     * 5. After task.run completes, we call afterExecute, which may
     * also throw an exception, which will also cause thread to
     * die. According to JLS Sec 14.20, this exception is the one that
     * will be in effect even if task.run throws.
     *
     * The net effect of the exception mechanics is that afterExecute
     * and the thread's UncaughtExceptionHandler have as accurate
     * information as we can provide about any problems encountered by
     * user code.
     *
     * @param w the worker
     */
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 这就是线程池的每个线程都会一直循环从阻塞队列获取任务，
            // 初始化的时候，firstTask不为null，待核心线程数满了，就进入队列，一直从队列获取
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 检测是否已被线程池是否停止 或者当前 worker 被中断
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    // 执行线程的中断
                    wt.interrupt();
                try {
                    // 任务执行前的回调方法
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 注意看，这里是run而不是start，说明就用的这个线程本身执行，而不是另起新的线程，很重要，这就是线程复用的原因。
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 任务执行后的回调方法
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 任务执行退出清理操作
            processWorkerExit(w, completedAbruptly);
        }
    }
```

### 从队列获取任务

```

    /**
     * Performs blocking or timed wait for a task, depending on
     * current configuration settings, or returns null if this worker
     * must exit because of any of:
     * 1. There are more than maximumPoolSize workers (due to
     *    a call to setMaximumPoolSize).
     * 2. The pool is stopped.
     * 3. The pool is shutdown and the queue is empty.
     * 4. This worker timed out waiting for a task, and timed-out
     *    workers are subject to termination (that is,
     *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
     *    both before and after the timed wait, and if the queue is
     *    non-empty, this worker is not the last thread in the pool.
     *
     * @return task, or null if the worker must exit, in which case
     *         workerCount is decremented
     */
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 如果进行了 shutdown, 且队列为空, 则需要将 worker 退出
            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 线程数据大于最大允许线程，有超时机制的，需要删除多余的 Worker
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                // 带超时的获取，或者是阻塞的获取任务
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

### 停止线程池

```
    /**
     * Initiates an orderly shutdown in which previously submitted
     * tasks are executed, but no new tasks will be accepted.
     * Invocation has no additional effect if already shut down.
     *
     * <p>This method does not wait for previously submitted tasks to
     * complete execution.  Use {@link #awaitTermination awaitTermination}
     * to do that.
     *
     * @throws SecurityException {@inheritDoc}
     */
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 检查权限
            checkShutdownAccess();
            // 设置运行状态为 SHUTDOWN
            advanceRunState(SHUTDOWN);
            // 中断线程
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        // 没有强制设置状态，而是如果还有待执行的任务，继续执行，如果没有待执行的任务，就设置 TERMINATED 状态
        tryTerminate();
    }


    /**
     * Attempts to stop all actively executing tasks, halts the
     * processing of waiting tasks, and returns a list of the tasks
     * that were awaiting execution. These tasks are drained (removed)
     * from the task queue upon return from this method.
     *
     * <p>This method does not wait for actively executing tasks to
     * terminate.  Use {@link #awaitTermination awaitTermination} to
     * do that.
     *
     * <p>There are no guarantees beyond best-effort attempts to stop
     * processing actively executing tasks.  This implementation
     * cancels tasks via {@link Thread#interrupt}, so any task that
     * fails to respond to interrupts may never terminate.
     *
     * @throws SecurityException {@inheritDoc}
     */
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            // 设置运行状态为 STOP
            advanceRunState(STOP);
            interruptWorkers();
            // 注意：这里与shutdown的区别，它会清空队列，待处理任务都没法继续执行
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }

```



## 参考

1. <a href="https://www.cnblogs.com/sanzao/p/10712778.html" target="_blank">并发系列（6）之 ThreadPoolExecutor 详解</a>
1. <a href="https://www.pdai.tech/md/java/thread/java-thread-x-juc-executor-ThreadPoolExecutor.html" target="_blank">JUC线程池: ThreadPoolExecutor详解</a>
1. <a href="https://www.cnblogs.com/yougewe/p/12267274.html" target="_blank">线程池技术之：ThreadPoolExecutor 源码解析</a>









