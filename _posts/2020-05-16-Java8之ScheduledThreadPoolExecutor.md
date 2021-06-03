---
layout:     post
title:      Java8之ScheduledThreadPoolExecutor
subtitle:   调度线程池
date:       2020-05-16
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java
    - ScheduledThreadPoolExecutor
    - ThreadPoolExecutor

---
# 注意
> 想法及时记录，实现可以待做。

## 简介

线程池解决了两个不同的问题：由于减少了每个任务的调用开销，它们通常在执行大量异步任务时提供更好的性能。

并且它们提供了一种约束和管理资源的方法，包括线程在执行任务集合时消耗的资源。每个任务还维护一些基本的统计数据，如已完成的任务数量。

使用线程池的好处有：
- 可管理：线程属于稀缺资源，不能无限制创建，否则会造成系统的稳定性。使用线程池可以进行统一的分配、调优和监控。
- 性能提升：创建线程是个很大的开销，能复用线程对于性能的提升有帮助。

有时候业务会需要周期性的执行任务，那么一般的线程池是无法满足这种需求的，此时，能周期性的执行的线程池就很有需要了。

ThreadPoolExecutor的UML类图如图所示：

![ThreadPoolExecutor]({{site.url}}/img/java/ThreadPoolExecutor.png)


### 组成介绍

```
    /**
     * 用于标识线程池shutdown时，是否继续执行已经存在于任务队列中的定时任务
     * False if should cancel/suppress periodic tasks on shutdown.
     */
    private volatile boolean continueExistingPeriodicTasksAfterShutdown;

    /**
     * 用于标识线程池shutdown时，是否继续执行已经存在于任务队列中的定时任务
     * False if should cancel non-periodic tasks on shutdown.
     */
    private volatile boolean executeExistingDelayedTasksAfterShutdown = true;

    /**
     * 用于标识如果当前任务已经取消了，是否将其从任务队列中真正的移除，而不只是标识其为删除状态
     * True if ScheduledFutureTask.cancel should remove from queue
     */
    private volatile boolean removeOnCancel = false;

    /**
     * 其为一个AtomicLong类型的变量，该变量记录了当前任务被创建时是第几个任务的一个序号，
     * 这个序号的主要用于确认当两个任务开始执行时间相同时具体哪个任务先执行，
     * 比如两个任务的开始执行时间都为1515847881158，那么序号小的任务将先执行
     * Sequence number to break scheduling ties, and in turn to
     * guarantee FIFO order among tied entries.
     */
    private static final AtomicLong sequencer = new AtomicLong();

```

### 类的介绍

ScheduledThreadPoolExecutor继承了ThreadPoolExecutor,实现了ScheduledExecutorService，这个调度线程池拥有ThreadPoolExecutor的特性。

ScheduledExecutorService有四个接口，一类是一定延迟后执行一次，一类是一定延迟后周期性执行。

```
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService{}

    // 给定核心线程数、队列容量非常大，阻塞队列是 延迟队列，这个队列专用调度线程池使用
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }

    // 指定延迟后，执行一次任务
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);

    // 指定延迟后，执行一次任务，有返回值
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);
    // 初始延迟后，周期性执行任务，固定频率的执行。
    // initialDelay后一次，initialDelay + 1 * period，initialDelay + 2 * period ...
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);
    // 初始延迟后，周期性执行任务，固定延迟的执行。一次执行结束后 + delay = 下一次开始执行。
    // initialDelay后一次，initialDelay + 1 * period，initialDelay + 2 * period ... 
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);


```

### 关键内部类

```
    // 执行的任务进行了封装
    private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {

        // 序列号
        /** Sequence number to break ties FIFO */
        private final long sequenceNumber;

        // 任务执行的时间，单位是纳秒
        /** The time the task is enabled to execute in nanoTime units */
        private long time;

        /**
         * 纳秒的时间间隔，正数表明是 fixed-rate固定频率的周期任务，负数表明是 fixed-delay 固定延迟的周期任务，0表示不需要重复，执行一次的任务
         * Period in nanoseconds for repeating tasks.  A positive
         * value indicates fixed-rate execution.  A negative value
         * indicates fixed-delay execution.  A value of 0 indicates a
         * non-repeating task.
         */
        private final long period;

        // 周期性执行任务
        /** The actual task to be re-enqueued by reExecutePeriodic */
        RunnableScheduledFuture<V> outerTask = this;

        /**
         * 当前任务在延迟队列数组中的下标
         * Index into delay queue, to support faster cancellation.
         */
        int heapIndex;

        // 任务之间的比较，先比较执行时间，时间小的先执行，如果时间一样，那就在比较序列号，序列号小的先执行
        public int compareTo(Delayed other) {
            if (other == this) // compare zero if same object
                return 0;
            if (other instanceof ScheduledFutureTask) {
                ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                long diff = time - x.time;
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
                else if (sequenceNumber < x.sequenceNumber)
                    return -1;
                else
                    return 1;
            }
            long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
            return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
        }

        /**
         * Returns {@code true} if this is a periodic (not a one-shot) action.
         *
         * @return {@code true} if periodic
         */
        public boolean isPeriodic() {
            return period != 0;
        }

        /**
         * 周期性任务的下次执行时间
         * period 大于0，是 fixed-rate，也即固定频率，下次的时间就是本次的执行开始时间+时间间隔
         * period 小于0，是 fixed-delay，也即固定延迟，下次的时间就是当前时间+时间间隔
         * Sets the next time to run for a periodic task.
         */
        private void setNextRunTime() {
            long p = period;
            if (p > 0)
                time += p;
            else
                time = triggerTime(-p);
        }

        // 取消任务的执行，如果true，则立即中断执行，如果是false，则等待其执行完成后在取消。
        public boolean cancel(boolean mayInterruptIfRunning) {
            boolean cancelled = super.cancel(mayInterruptIfRunning);
            if (cancelled && removeOnCancel && heapIndex >= 0)
                remove(this);
            return cancelled;
        }

        /**
         * Overrides FutureTask version so as to reset/requeue if periodic.
         */
        public void run() {
            boolean periodic = isPeriodic();
            // 线程池状态不适合周期性任务，那么就取消执行
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            // 如果线程池状态可以执行任务，但是不是周期性任务，那就立即用当前线程执行一次
            else if (!periodic)
                ScheduledFutureTask.super.run();
            // 如果线程池状态可以执行周期性任务，立即执行一次任务，然后重置任务的状态。
            // 如果成功，就更新下一次的执行时间，将任务再次入队等待下一次执行。
            else if (ScheduledFutureTask.super.runAndReset()) {
                setNextRunTime();
                reExecutePeriodic(outerTask);
            }
        }
    }

    /**
     * DelayedWorkQueue的实现与DelayQueue以及PriorityQueue的实现基本相似，形式都为一个优先队列，并且底层是使用堆结构来实现优先队列的功能，在数据存储方式上，其使用的是数组来实现。
     * DelayedWorkQueue中主要存储ScheduledFutureTask类型的任务，该任务中有一个heapIndex属性保存了当前任务在当前队列数组中的位置下标，其主要提升的是对队列的诸如contains()和remove()等需要定位当前任务位置的方法的效率，时间复杂度可以从O(N)提升到O(logN)。
     * Specialized delay queue. To mesh with TPE declarations, this
     * class must be declared as a BlockingQueue<Runnable> even though
     * it can only hold RunnableScheduledFutures.
     */
    static class DelayedWorkQueue extends AbstractQueue<Runnable>
        implements BlockingQueue<Runnable> {

        /*
         * A DelayedWorkQueue is based on a heap-based data structure
         * like those in DelayQueue and PriorityQueue, except that
         * every ScheduledFutureTask also records its index into the
         * heap array. This eliminates the need to find a task upon
         * cancellation, greatly speeding up removal (down from O(n)
         * to O(log n)), and reducing garbage retention that would
         * otherwise occur by waiting for the element to rise to top
         * before clearing. But because the queue may also hold
         * RunnableScheduledFutures that are not ScheduledFutureTasks,
         * we are not guaranteed to have such indices available, in
         * which case we fall back to linear search. (We expect that
         * most tasks will not be decorated, and that the faster cases
         * will be much more common.)
         *
         * All heap operations must record index changes -- mainly
         * within siftUp and siftDown. Upon removal, a task's
         * heapIndex is set to -1. Note that ScheduledFutureTasks can
         * appear at most once in the queue (this need not be true for
         * other kinds of tasks or work queues), so are uniquely
         * identified by heapIndex.
         */
        
        // 数组初始化容量大小
        private static final int INITIAL_CAPACITY = 16;
        // 存储数组
        private RunnableScheduledFuture<?>[] queue =
            new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
        // 添加和删除的时候锁
        private final ReentrantLock lock = new ReentrantLock();
        // 队列中的有效任务个数
        private int size = 0;

        /**
         * Thread designated to wait for the task at the head of the
         * queue.  This variant of the Leader-Follower pattern
         * (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to
         * minimize unnecessary timed waiting.  When a thread becomes
         * the leader, it waits only for the next delay to elapse, but
         * other threads await indefinitely.  The leader thread must
         * signal some other thread before returning from take() or
         * poll(...), unless some other thread becomes leader in the
         * interim.  Whenever the head of the queue is replaced with a
         * task with an earlier expiration time, the leader field is
         * invalidated by being reset to null, and some waiting
         * thread, but not necessarily the current leader, is
         * signalled.  So waiting threads must be prepared to acquire
         * and lose leadership while waiting.
         */
        // 执行队列头部任务的线程
        private Thread leader = null;

        /**
         * Condition signalled when a newer task becomes available at the
         * head of the queue or a new thread may need to become leader.
         */
        private final Condition available = lock.newCondition();

        /**
         * Sets f's heapIndex if it is a ScheduledFutureTask.
         */
        private void setIndex(RunnableScheduledFuture<?> f, int idx) {
            if (f instanceof ScheduledFutureTask)
                ((ScheduledFutureTask)f).heapIndex = idx;
        }

        /**
         * Sifts element added at bottom up to its heap-ordered spot.
         * Call only when holding lock.
         */
        private void siftUp(int k, RunnableScheduledFuture<?> key) {
            while (k > 0) {
                int parent = (k - 1) >>> 1;
                RunnableScheduledFuture<?> e = queue[parent];
                if (key.compareTo(e) >= 0)
                    break;
                queue[k] = e;
                setIndex(e, k);
                k = parent;
            }
            queue[k] = key;
            setIndex(key, k);
        }

        /**
         * Sifts element added at top down to its heap-ordered spot.
         * Call only when holding lock.
         */
        private void siftDown(int k, RunnableScheduledFuture<?> key) {
            int half = size >>> 1;
            while (k < half) {
                int child = (k << 1) + 1;
                RunnableScheduledFuture<?> c = queue[child];
                int right = child + 1;
                if (right < size && c.compareTo(queue[right]) > 0)
                    c = queue[child = right];
                if (key.compareTo(c) <= 0)
                    break;
                queue[k] = c;
                setIndex(c, k);
                k = child;
            }
            queue[k] = key;
            setIndex(key, k);
        }

        /**
         * 扩容50%
         * Resizes the heap array.  Call only when holding lock.
         */
        private void grow() {
            int oldCapacity = queue.length;
            int newCapacity = oldCapacity + (oldCapacity >> 1); // grow 50%
            if (newCapacity < 0) // overflow
                newCapacity = Integer.MAX_VALUE;
            queue = Arrays.copyOf(queue, newCapacity);
        }

        /**
         * Finds index of given object, or -1 if absent.
         */
        private int indexOf(Object x) {
            if (x != null) {
                if (x instanceof ScheduledFutureTask) {
                    int i = ((ScheduledFutureTask) x).heapIndex;
                    // Sanity check; x could conceivably be a
                    // ScheduledFutureTask from some other pool.
                    if (i >= 0 && i < size && queue[i] == x)
                        return i;
                } else {
                    for (int i = 0; i < size; i++)
                        if (x.equals(queue[i]))
                            return i;
                }
            }
            return -1;
        }

        public boolean remove(Object x) {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                // 没找到值，移除失败
                int i = indexOf(x);
                if (i < 0)
                    return false;
                
                // 这个值的下标给-1
                setIndex(queue[i], -1);
                int s = --size;
                // 获取最后一个位置，这个位置可以置为null，因为这个位置没得元素了，需要调整
                RunnableScheduledFuture<?> replacement = queue[s];
                queue[s] = null;
                // 被移除的不是最后一个元素，那么需要调整结构
                if (s != i) {
                    siftDown(i, replacement);
                    if (queue[i] == replacement)
                        siftUp(i, replacement);
                }
                return true;
            } finally {
                lock.unlock();
            }
        }

        // 获取队首元素
        public RunnableScheduledFuture<?> peek() {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                return queue[0];
            } finally {
                lock.unlock();
            }
        }

        public boolean offer(Runnable x) {
            if (x == null)
                throw new NullPointerException();
            RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                int i = size;
                // 容量不够新增数据了，需要扩容，50%
                if (i >= queue.length)
                    grow();
                size = i + 1;
                // 队列初始化
                if (i == 0) {
                    queue[0] = e;
                    setIndex(e, 0);
                } else {
                    // 调整堆结构
                    siftUp(i, e);
                }
                // 入队的是队首数据
                if (queue[0] == e) {
                    leader = null;
                    available.signal();
                }
            } finally {
                lock.unlock();
            }
            return true;
        }


        /**
         * Performs common bookkeeping for poll and take: Replaces
         * first element with last and sifts it down.  Call only when
         * holding lock.
         * @param f the task to remove and return
         */
        private RunnableScheduledFuture<?> finishPoll(RunnableScheduledFuture<?> f) {
            int s = --size;
            RunnableScheduledFuture<?> x = queue[s];
            queue[s] = null;
            if (s != 0)
                siftDown(0, x);
            setIndex(f, -1);
            return f;
        }

        public RunnableScheduledFuture<?> poll() {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                // 队列为空，或者延迟还没到时间，那么返回为null
                RunnableScheduledFuture<?> first = queue[0];
                if (first == null || first.getDelay(NANOSECONDS) > 0)
                    return null;
                else
                    // 返回队首数据
                    return finishPoll(first);
            } finally {
                lock.unlock();
            }
        }

        public RunnableScheduledFuture<?> take() throws InterruptedException {
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                    // 队列为空，则当前线程进入等待
                    RunnableScheduledFuture<?> first = queue[0];
                    if (first == null)
                        available.await();
                    else {
                        // 延时到了，则返回队首数据
                        long delay = first.getDelay(NANOSECONDS);
                        if (delay <= 0)
                            return finishPoll(first);
                        // 有线程等待队首数据，则当前线程进入等待
                        first = null; // don't retain ref while waiting
                        if (leader != null)
                            available.await();
                        else {
                            // 当前线程作为队首数据的处理线程，然后继续等待延时后获取任务执行
                            Thread thisThread = Thread.currentThread();
                            leader = thisThread;
                            try {
                                available.awaitNanos(delay);
                            } finally {
                                if (leader == thisThread)
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
                // 队首线程为null，队列还有数据，那么可以唤醒等待的线程执行任务
                if (leader == null && queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }

        public RunnableScheduledFuture<?> poll(long timeout, TimeUnit unit)
            throws InterruptedException {
            long nanos = unit.toNanos(timeout);
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                    RunnableScheduledFuture<?> first = queue[0];
                    if (first == null) {
                        if (nanos <= 0)
                            return null;
                        else
                            nanos = available.awaitNanos(nanos);
                    } else {
                        long delay = first.getDelay(NANOSECONDS);
                        if (delay <= 0)
                            return finishPoll(first);
                        if (nanos <= 0)
                            return null;
                        first = null; // don't retain ref while waiting
                        if (nanos < delay || leader != null)
                            nanos = available.awaitNanos(nanos);
                        else {
                            Thread thisThread = Thread.currentThread();
                            leader = thisThread;
                            try {
                                long timeLeft = available.awaitNanos(delay);
                                nanos -= delay - timeLeft;
                            } finally {
                                if (leader == thisThread)
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
                if (leader == null && queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }


        public int drainTo(Collection<? super Runnable> c) {
            if (c == null)
                throw new NullPointerException();
            if (c == this)
                throw new IllegalArgumentException();
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                RunnableScheduledFuture<?> first;
                int n = 0;
                while ((first = peekExpired()) != null) {
                    c.add(first);   // In this order, in case add() throws.
                    finishPoll(first);
                    ++n;
                }
                return n;
            } finally {
                lock.unlock();
            }
        }

        public int drainTo(Collection<? super Runnable> c, int maxElements) {
            if (c == null)
                throw new NullPointerException();
            if (c == this)
                throw new IllegalArgumentException();
            if (maxElements <= 0)
                return 0;
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                RunnableScheduledFuture<?> first;
                int n = 0;
                while (n < maxElements && (first = peekExpired()) != null) {
                    c.add(first);   // In this order, in case add() throws.
                    finishPoll(first);
                    ++n;
                }
                return n;
            } finally {
                lock.unlock();
            }
        }

    }

```

### schedule方法

```


    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        // 返回ScheduledFutureTask 类型的封装
        RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
        // 延迟执行任务
        delayedExecute(t);
        return t;
    }

    /**
     * Main execution method for delayed or periodic tasks.  If pool
     * is shut down, rejects the task. Otherwise adds task to queue
     * and starts a thread, if necessary, to run it.  (We cannot
     * prestart the thread to run the task because the task (probably)
     * shouldn't be run yet.)  If the pool is shut down while the task
     * is being added, cancel and remove it if required by state and
     * run-after-shutdown parameters.
     *
     * @param task the task
     */
    private void delayedExecute(RunnableScheduledFuture<?> task) {
        // 线程池状态 不在运行中，则TPE的拒绝策略
        if (isShutdown())
            reject(task);
        else {
            // 线程池状态为 运行中，那么将任务入队
            super.getQueue().add(task);
            // 再次验证线程池的状态，如果不可用，就移除任务
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
                // 里面的逻辑就是，要么添加核心线程，要么非核心线程，保证有线程去执行任务
                ensurePrestart();
        }
    }

```

### scheduleAtFixedRate方法

```

    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        // 封装任务,构造方法注意看 时间 是怎么算的。正数，是固定频率。
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        // 这里的装饰器其实是给扩展使用，本类没做变化
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        // 
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }

```


### scheduleWithFixedDelay方法

```

    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        // 封装任务,构造方法注意看 时间 是怎么算的。负数，是固定延迟。
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }

```


## 参考

1. <a href="https://www.cnblogs.com/sanzao/p/10760641.html" target="_blank">并发系列（7）之 ScheduledThreadPoolExecutor 详解</a>
2. <a href="https://www.jianshu.com/p/203b28223c70" target="_blank">ScheduledThreadPoolExecutor详解</a>









