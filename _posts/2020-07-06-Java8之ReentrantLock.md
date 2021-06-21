---
layout:     post
title:      Java8之ReentrantLock
subtitle:   可重入锁
date:       2020-07-06
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java
    - ReentrantLock
    - Lock
    - AbstractQueuedSynchronizer
    - AQS
    - JUC

---
# 注意
> 想法及时记录，实现可以待做。

## 简介
ReentrantLock是JUC并发工具包中提供的可中断、可重入、允许超时的一种锁。

它的实现主要依赖AQS机制，是AQS的一个实际应用，对于AQS的分析，详见<a href="https://qin4zhang.github.io/2020/06/06/Java8%E4%B9%8BAbstractQueuedSynchronizer/" target="_blank">Java8之AbstractQueuedSynchronizer</a>

相比较Synchronized的使用锁方式，两者的对比如下：

1. 一个是属于代码api的调用，一个是依赖JVM
2. 一个是显示加锁、释放锁，一个是通过关键字隐式的加锁、释放锁，后者相比较更简单
3. api的调用，无疑更加灵活，可以有线程中断，可以超时机制，而关键字缺少这种可控性、灵活性
4. 都可以重入

||ReentrantLock|Synchronized|
|---|---|---|
|锁实现机制|依赖AQS|JVM层的monitor|
|使用方式|lock、unlock方法调用|Synchronized隐式加锁、释放锁|

### 组成介绍


```

    // 锁的实现
    private final Sync sync;

    /**
     * 默认构造器，非公平锁
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * fair为true，则构造公平锁，否则是非公平锁
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }


```

### 关键内部类


```

    /**
     * sync锁的抽象静态内部类，继承AQS实现，state个数表示持有锁的数量
     * 子类有公平和非公平方式
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * 具体子类实现抽象方法，模板模式的应用
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        abstract void lock();

        /**
         * 非公平的方式获取锁
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            // 锁还没有被其他线程获取，当前线程尝试获取，如果成功了，那么就直接返回成功，不用入队等待。
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 锁已经被其他线程获取，判断是不是当前线程获取的，毕竟同一个线程可以重复获取锁。
            // 如果是当前线程重复获取锁，那么锁的次数累加即可
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            // 锁被其他线程获取，当前线程只能入队等待，获取失败。
            return false;
        }

        // 释放锁，公平锁和非公平锁的共同逻辑
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            // 当前线程是持有锁的线程才可以，否则抛异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            // 有可能是重入锁，当前线程还会持有锁，此时并不是完全释放
            boolean free = false;
            // 如果为0，表示这次释放锁，线程已经完全释放，而不会重入很多次，一层一层的释放。
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            // AQS.state的值
            setState(c);
            return free;
        }

        // 当前线程是否持有锁
        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class

        // 持有锁的线程
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        // 持有锁的计数次数
        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        // 锁是否被占用
        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }

    /**
     * 非公平锁的实现
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * 非公平锁模式，获取锁的方式是，先cas尝试获取，万一成功了，那么当前线程就直接获取锁，否则就按照正常的流程去使用AQS.acquire获取锁
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        // AQS中的这个方法，由具体子类实现。在上述acquire的时候会使用
        protected final boolean tryAcquire(int acquires) {
            // 这个方法是 Sync.nonfairTryAcquire，可以看前文的分析
            return nonfairTryAcquire(acquires);
        }
    }

    /**
     * 公平锁的实现
     * 锁没有被占用，等待队列没有其他线程等待锁则尝试cas获取锁；否则如果是当前线程，则重入锁即可，否则获取失败
     * 公平锁的实现
     * Sync object for fair locks
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;
        
        // 公平方式的获取锁，那就是按照正常的流程去使用AQS.acquire获取锁，不会首先cas尝试是否能获取到锁
        final void lock() {
            acquire(1);
        }

        /**
         * AQS中的这个方法，由具体子类实现。在上述acquire的时候会使用
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            // 锁还没有被线程获取
            if (c == 0) {
                // 如果等待队列没有线程等待获取，那么当前线程尝试cas获取锁。如果成功，那么说明当前线程可以占用这个锁，也就是获取成功。
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 锁被占用，如果是当前线程自己占用的，那么就重入，增加锁的计数器次数
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            // 其他线程占用锁，则当前线程抢锁失败
            return false;
        }
    }


```


## 参考

1. <a href="https://www.jianshu.com/p/3f3417dbcac4" target="_blank">ReentrantLock 源码分析 (基于Java 8)</a>
2. <a href="https://segmentfault.com/a/1190000022732139" target="_blank">AQS之ReentrantLock源码解析</a>









