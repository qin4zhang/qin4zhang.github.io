---
layout:     post
title:      Java8之AbstractQueuedSynchronizer
subtitle:   线程安全
date:       2020-06-06
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java
    - AbstractQueuedSynchronizer
    - AQS
    - JUC

---
# 注意
> 想法及时记录，实现可以待做。

## 简介

AbstractQueuedSynchronizer 这个类可以说是JUC这个并发包里面非常重要的一个基础框架类了，在它的基础上，实现了JUC里面的很多特性的并发工具，比如 ReentrantLock、CountDownLatch等等。



### 组成介绍

AQS内部维护一个FIFO的队列，这个队列是CLH队列的一种实现形式，队列中的元素就是Node节点。

```
    /**
     * 队列的头节点，看这关键字，肯定有cas操作
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
    private transient volatile Node head;

    /**
     * 队列的尾节点，看这关键字，肯定有cas操作
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
    private transient volatile Node tail;

    /**
     * 同步器的状态，看这关键字，肯定有cas操作
     * The synchronization state.
     */
    private volatile int state;


```

### 关键内部类

等待队列的节点类，等待队列是CLH（Craig,Landin,Hagersten）锁队列的一种变体，CLH锁通常用来作为自旋锁。

```
    static final class Node {

        // 共享模式的节点
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();

        // 独占模式的节点
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;

        // 节点的状态值
        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;

        /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
        volatile int waitStatus;

        /**
         * 前驱节点
         * Link to predecessor node that current node/thread relies on
         * for checking waitStatus. Assigned during enqueuing, and nulled
         * out (for sake of GC) only upon dequeuing.  Also, upon
         * cancellation of a predecessor, we short-circuit while
         * finding a non-cancelled one, which will always exist
         * because the head node is never cancelled: A node becomes
         * head only as a result of successful acquire. A
         * cancelled thread never succeeds in acquiring, and a thread only
         * cancels itself, not any other node.
         */
        volatile Node prev;

        /**
         * 后继节点
         * Link to the successor node that the current node/thread
         * unparks upon release. Assigned during enqueuing, adjusted
         * when bypassing cancelled predecessors, and nulled out (for
         * sake of GC) when dequeued.  The enq operation does not
         * assign next field of a predecessor until after attachment,
         * so seeing a null next field does not necessarily mean that
         * node is at end of queue. However, if a next field appears
         * to be null, we can scan prev's from the tail to
         * double-check.  The next field of cancelled nodes is set to
         * point to the node itself instead of null, to make life
         * easier for isOnSyncQueue.
         */
        volatile Node next;

        /**
         * 节点入队时的线程
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        volatile Thread thread;

        /**
         * condition队列的后继节点
         * Link to next node waiting on condition, or the special
         * value SHARED.  Because condition queues are accessed only
         * when holding in exclusive mode, we just need a simple
         * linked queue to hold nodes while they are waiting on
         * conditions. They are then transferred to the queue to
         * re-acquire. And because conditions can only be exclusive,
         * we save a field by using special value to indicate shared
         * mode.
         */
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * 返回前驱节点
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }

```

### 入队

1. 队列为空，则初始化队列，保证头节点和尾节点都初始化完成
2. 入队的新节点插入队列的尾部，几个指向指针都需要修改

```

    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
    private Node enq(final Node node) {
        // 看到这里就可以猜到，会有cas操作出现
        for (;;) {
            // 尾节点为null，说明队列为空，先初始化队列
            Node t = tail;
            if (t == null) { // Must initialize
                // 新的节点给到头节点，如果成功，那么尾节点和头节点一样
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                // 队列不为空
                // 入队的新节点的前驱节点，就是尾节点，也就是新的节点插入队列的尾部
                node.prev = t;
                // 将新节点cas操作到尾节点上，这样就完成新节点添加至队列尾部，如果成功，那么尾节点的下一个指向新节点，最后返回尾节点
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

    /**
     * 添加的节点，要么是独占节点，要么是共享节点，返回封装后的新节点
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        // 当前线程和给定的节点，封装成新的节点，然后插入队列尾部
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        // 尾节点不为null，则尝试一次cas，将新节点插入到队列的尾部，如果失败，那就按照自旋+cas的入队方式入队节点
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

```


### 独占式获取锁

```
    /**
     * 独占模式的获取，忽略中断。
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {
        // tryAcquire这个方法由子类实现
        if (!tryAcquire(arg) &&
            // 如果获取失败，那么通过 addWaiter将这个独占节点插入队列尾部,再通过 acquireQueued 不断尝试获取锁
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            // 由于不响应中断，如果检测到中断，acquireQueued() 会返回 true，进入方法体
            // 由于检测时使用了 Thread.interrupted()，中断标志被重置，需要恢复中断标志
            selfInterrupt();
    }

    /**
     * 独占模式的获取，如果有中断，就终止获取
     * Acquires in exclusive mode, aborting if interrupted.
     * Implemented by first checking interrupt status, then invoking
     * at least once {@link #tryAcquire}, returning on
     * success.  Otherwise the thread is queued, possibly repeatedly
     * blocking and unblocking, invoking {@link #tryAcquire}
     * until success or the thread is interrupted.  This method can be
     * used to implement method {@link Lock#lockInterruptibly}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @throws InterruptedException if the current thread is interrupted
     */
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        // 线程中断，则终止获取，抛异常
        if (Thread.interrupted())
            throw new InterruptedException();
        // 子类实现的方法，如果获取失败，那么尝试中断方式的获取锁
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }


    /**
     * 在独占模式，非中断下从队列获取线程
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // 节点的前驱结点
                final Node p = node.predecessor();
                // 如果前驱结点是头节点，并且获取锁成功，那么头节点是获取到锁，所以这个节点可以设置为头节点，然后返回没有中断
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 获取失败后，查看是否需要被挂起，如果需要挂起，检查是否有中断信息
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    /**
     * 独占模式 有中断的获取锁。跟不中断的对比，大体差不多。唯一不同在于，如果有中断，则终止获取，抛出异常。
     * Acquires in exclusive interruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }



    /**
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev.
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        // 前驱节点的状态为 SIGNAL ,说明当前节点，等着前驱节点释放锁或者取消了，再唤醒
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        // 说明前驱节点的状态为 CANCELLED == 1 ,是取消的状态，那么就跳过这些取消状态的节点，一直往前遍历，直到不是 取消的节点
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * 前驱节点的状态 一定是 0 或者 PROPAGATE 状态
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }


```

### 共享式获取锁

```

    /**
     * 共享模式的获取锁，忽略中断
     * Acquires in shared mode, ignoring interrupts.  Implemented by
     * first invoking at least once {@link #tryAcquireShared},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquireShared} until success.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquireShared} but is otherwise uninterpreted
     *        and can represent anything you like.
     */
    public final void acquireShared(int arg) {
        // 子类实现
        // 返回值小于0，表示获取失败
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    /**
     * 共享模式的获取，如果有中断，就终止获取
     * Acquires in shared mode, aborting if interrupted.  Implemented
     * by first checking interrupt status, then invoking at least once
     * {@link #tryAcquireShared}, returning on success.  Otherwise the
     * thread is queued, possibly repeatedly blocking and unblocking,
     * invoking {@link #tryAcquireShared} until success or the thread
     * is interrupted.
     * @param arg the acquire argument.
     * This value is conveyed to {@link #tryAcquireShared} but is
     * otherwise uninterpreted and can represent anything
     * you like.
     * @throws InterruptedException if the current thread is interrupted
     */
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        // 线程中断，终止获取，抛异常
        if (Thread.interrupted())
            throw new InterruptedException();
        // 获取失败
        if (tryAcquireShared(arg) < 0)
            // 跟doAcquireShared 差不多，只是会存在线程中断，终止获取，抛异常
            doAcquireSharedInterruptibly(arg);
    }

    /**
     * 共享、不中断获取锁
     * Acquires in shared uninterruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireShared(int arg) {
        // 入队共享节点，然后封装成带有线程的新节点
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // 新节点的前驱节点，如果是头节点，再次尝试获取
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        // 获取成功，设置头节点，往后传播唤醒后继节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        // 判断中断
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                // 不是头节点，获取失败，线城是否需要挂起，再次检查线程中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    /**
     * 设置队列的头节点
     * Sets head of queue, and checks if successor may be waiting
     * in shared mode, if so propagating if either propagate > 0 or
     * PROPAGATE status was set.
     *
     * @param node the node
     * @param propagate the return value from a tryAcquireShared
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        // 头节点
        Node h = head; // Record old head for check below
        // 设置新的头节点
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        // 
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            // 头节点的下一个节点，如果为null，或者是共享节点
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }







```



### 独占模式释放锁

```


    /**
     * Releases in exclusive mode.  Implemented by unblocking one or
     * more threads if {@link #tryRelease} returns true.
     * This method can be used to implement method {@link Lock#unlock}.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryRelease} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @return the value returned from {@link #tryRelease}
     */
    public final boolean release(int arg) {
        // 子类释放锁成功
        if (tryRelease(arg)) {
            Node h = head;
            // 头节点不为null，并且等待状态不为0.那么唤醒后继节点
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    /**
     * 唤醒节点的后继节点
     * Wakes up node's successor, if one exists.
     *
     * @param node the node
     */
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        // 获取节点的状态，如果状态不是初始状态或cancelled，则设置为初始态
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        // 后继节点
        Node s = node.next;
        // 后继节点如果为null，或者 状态为取消
        if (s == null || s.waitStatus > 0) {
            s = null;
            // 从队列尾部遍历需要通知的最近的节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        // 后继节点如果不为null，那么唤醒后继节点
        if (s != null)
            LockSupport.unpark(s.thread);
    }



```

### 共享模式释放锁

```

    /**
     * 共享模式释放锁
     * Releases in shared mode.  Implemented by unblocking one or more
     * threads if {@link #tryReleaseShared} returns true.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryReleaseShared} but is otherwise uninterpreted
     *        and can represent anything you like.
     * @return the value returned from {@link #tryReleaseShared}
     */
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

    /**
     * Release action for shared mode -- signals successor and ensures
     * propagation. (Note: For exclusive mode, release just amounts
     * to calling unparkSuccessor of head if it needs signal.)
     */
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            // 头节点不为null，并且头节点与尾节点不一样
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                // 头节点的等待状态如果是 SIGNAL,则将后继节点唤醒
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                // 
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            // 头节点有变化，则不需要唤醒
            if (h == head)                   // loop if head changed
                break;
        }
    }


```




## 参考

1. <a href="http://gee.cs.oswego.edu/dl/papers/aqs.pdf" target="_blank">The java.util.concurrent Synchronizer Framework</a>
2. <a href="http://ifeve.com/introduce-abstractqueuedsynchronizer/" target="_blank">AbstractQueuedSynchronizer的介绍和原理分析</a>
3. <a href="https://juejin.cn/post/6844903939314302984" target="_blank">超详细！AQS（AbstractQueuedSynchronizer）源码解析</a>
4. <a href="https://segmentfault.com/a/1190000014221325" target="_blank">源码分析JDK8之AbstractQueuedSynchronizer</a>
5. <a href="https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html" target="_blank">从ReentrantLock的实现看AQS的原理及应用</a>









