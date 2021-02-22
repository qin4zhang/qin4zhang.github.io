---
layout:     post
title:      Java8之ConcurrentHashMap介绍
subtitle:   ConcurrentHashMap
date:       2020-03-28
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java8
    - ConcurrentHashMap
    - Map

---
# 注意
> 想法及时记录，实现可以待做。

## Map

Map常用有哪些呢？如图所示：

![Map]({{site.url}}/img/java/Java8-Map-Diagram.png)

## 简介
HashMap是Java中用于映射(键值对)处理的数据类型。但是不能用于多线程的情况下，否则会出现线程安全的问题。那么多线程下用哪个呢？

ConcurrentHashMap便是最好的选择了，由Doug Lea推出来的并发编程包的一员。

HashMap允许键和值为null，但是ConcurrentHashMap不允许键和值为null，否则NPE伺候。

它的复杂度一般来说是常数级，也就是O(1)，但是也不绝对，如果有hash碰撞了，有可能退化为O(n)或者是O(log n)。

如果不熟悉HashMap，可以看![Java8之HashMap介绍](./2020-03-21-Java8之HashMap介绍.md)。

## 分析
ConcurrentHashMap在Java8中的存储结构与HashMap是一样的，也是数组+链表+红黑树。与HashMap很多类似的情况我们就不做太多的分析，就看看是怎么做到保证线程安全的吧。

### 组成介绍

```
    /**
     * 桶数组，第一次插入的时候才会初始化，内存可见性保证值在其他线程中都是最新的
     * The array of bins. Lazily initialized upon first insertion.
     * Size is always a power of two. Accessed directly by iterators.
     */
    transient volatile Node<K,V>[] table;

    /**
     * 初始化和控制扩容。
     * -1：表示初始化
     * -(1 + n)：n表示活跃的扩容线程的个数。如果有-N，则有N-1个活跃的扩容线程。
     * Table initialization and resizing control.  When negative, the
     * table is being initialized or resized: -1 for initialization,
     * else -(1 + the number of active resizing threads).  Otherwise,
     * when table is null, holds the initial table size to use upon
     * creation, or 0 for default. After initialization, holds the
     * next element count value upon which to resize the table.
     */
    private transient volatile int sizeCtl;
```

### 关键的内部类

```

    /**
     * 存储的节点
     * Key-value entry.  This class is never exported out as a
     * user-mutable Map.Entry (i.e., one supporting setValue; see
     * MapEntry below), but can be used for read-only traversals used
     * in bulk tasks.  Subclasses of Node with a negative hash field
     * are special, and contain null keys and values (but are never
     * exported).  Otherwise, keys and vals are never null.
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        // 不可变的hash值，一旦Node初始化，该值不在允许改变。
        final int hash;
        // 不可变键
        final K key;
        // 这里用volatile修饰值，是因为多线程修改的情况下。保证值的内存可见性，数据一致。
        volatile V val;
        // 同样，保证Node被修改，其他线程能保证其内存可见性。
        volatile Node<K,V> next;

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }
    }

    /**
     * 红黑树的组成一部分
     * Nodes for use in TreeBins
     */
    static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;

        TreeNode(int hash, K key, V val, Node<K,V> next,
                 TreeNode<K,V> parent) {
            super(hash, key, val, next);
            this.parent = parent;
        }
    }

    /**
     * 红黑树的结构，封装了TreeNode
     * TreeNodes used at the heads of bins. TreeBins do not hold user
     * keys or values, but instead point to list of TreeNodes and
     * their root. They also maintain a parasitic read-write lock
     * forcing writers (who hold bin lock) to wait for readers (who do
     * not) to complete before tree restructuring operations.
     */
    static final class TreeBin<K,V> extends Node<K,V> {
        TreeNode<K,V> root;
        volatile TreeNode<K,V> first;
        volatile Thread waiter;
        volatile int lockState;
        // values for lockState
        static final int WRITER = 1; // set while holding write lock
        static final int WAITER = 2; // set when waiting for write lock
        static final int READER = 4; // increment value for setting read lock
    }

    /**
     * 扩容时候才会出现的节点，从构造函数可以看到，key,value,hash都是null,nextTable引用的是传入的数组
     * A node inserted at head of bins during transfer operations.
     */
    static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }
    }



```

### 查询

```

    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code key.equals(k)},
     * then this method returns {@code v}; otherwise it returns
     * {@code null}.  (There can be at most one such mapping.)
     *
     * @throws NullPointerException if the specified key is null
     */
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        // 获取实际分布的hash值
        int h = spread(key.hashCode());
        // 定位到查找的key对应的头节点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            // 头节点的hash值和键都是一致，表名找到了，返回值即可。
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // hash值小于0，说明是在扩容，根据ForwardingNode.find定位数据
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            // 链表或者红黑树，头节点的下一个不为空，继续遍历查找
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        // 没有查到，返回null
        return null;
    }

    // 键的hash值，高16位与低16位异或运算后，还要跟一个常量与运算。
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
    // 计算hash值的一个常量值，可以看到第一位是7，说明最高位是0，其他未全部是1.
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

    // 在hash桶的数组里通过下标取值，这里是用了Unsafe的方法
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

```

### 插入

流程为：
1. 计算键的hash值，由hash确定在桶数组的下标位置
2. 还没初始化的桶数组，需要进行初始化
3. 定位的hash桶的位置如果是null，说明没冲突，可以直接CAS尝试插入
4. 定位的hash桶的位置不为null，说明hash冲突了，如果头节点的hash值是正在扩容，说明正在扩容中
5. 如果节点的hash判断得知是链表，则循环遍历，要么尾节点新增，要么相同节点替换
6. 如果节点的类型是红黑树，则按照红黑树的方式插入新节点
7. 新节点插入完成，需要检查链表的长度，如果超过了阈值，则转为红黑树
8. 对当前数组容量做检查，如果超过了阈值，则扩容处理。

```

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        // 插入的键值为null，直接抛NPE异常
        if (key == null || value == null) throw new NullPointerException();
        // 获得键的hash值
        int hash = spread(key.hashCode());
        // 数组上的bin所表示的长度，如果没有hash冲突，那就是0，如果是链表或者红黑树，那就累计个数，决定这个节点是不是要树化
        int binCount = 0;
        // 一直循环 + CAS 尝试插入值
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 跟HashMap一个思路，都是第一次插入的时候进行初始化操作
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // 待插入的位置是空的，可以尝试CAS插入新值
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 直接cas操作插入数据，如果成功了，跳出循环；如果失败，说明有其他线程已经占用了这个位置，重新循环。
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 如果正在扩容
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            // hash冲突的处理
            else {
                V oldVal = null;
                // 利用synchronized同步锁住Node头节点
                synchronized (f) {
                    // 再次从数组的下标中获取数据，与之前的取值作比较，检查当前节点是否有变化，有变化进入下一轮自旋
                    // 因为不能保证，当前线程运行到这里，有没有其他线程对该节点进行修改
                    if (tabAt(tab, i) == f) {
                        // 头节点的hash值大于等于0，说明是链表结构
                        if (fh >= 0) {
                            // 链表的个数累计，下面的for循环遍历完成后，就知道这个bin所拥有的节点个数了。
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 链表中存在相同的Node，判断是否更新值即可
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                // 链表的尾节点插入新节点
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 头节点是红黑树结构，这里与HashMap不一样，CHM对TreeNode做了一定的封装
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    // 是否树化
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 对binCount做校验，如果超过了阈值，就需要扩容
        addCount(1L, binCount);
        return null;
    }
```

### 初始化

```
    /**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        // 这里是多线程的情况下，CAS的操作不一定成功，需要一直重试。如果重试成功，那么 table肯定不在为null。
        while ((tab = table) == null || tab.length == 0) {
            // 通过构造函数可以发现，sizeCtl这个值与cap容量大小一致，是 2的次幂的大小，但是数组一开始没有初始化
            // 如果发现 sizeCtl<0，说明有其他线程正在初始化，当前线程就不需要再次初始化。释放当前CPU的调度。
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // 没有其他线程初始化，则进行初始化操作，通过CAS保证成功，并且将 sizeCtl 设置为 -1，其他线程通过该变量，不会再次初始化
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        // 数组的初始化大小，这个值是通过构造函数给进来的
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // 重置sc, n >>> 2 也就是 n/4，所以 sc = n - n/4 = n * 3/4 = 0.75n
                        sc = n - (n >>> 2);
                    }
                } finally {
                    // 重置 sizeCtl这个控制变量
                    sizeCtl = sc;
                }
                // 不用再CAS一直自旋了，完成了逻辑，可以跳出了。
                break;
            }
        }
        return tab;
    }

```


### 有可能触发的扩容

```

    /**
     * Adds to count, and if table is too small and not already
     * resizing, initiates transfer. If already resizing, helps
     * perform transfer if work is available.  Rechecks occupancy
     * after a transfer to see if another resize is already needed
     * because resizings are lagging additions.
     *
     * @param x the count to add
     * @param check if <0, don't check resize, if <= 1 only check if uncontended
     */
    private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        // CAS更新 baseCount
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            // CAS 更新 baseCount失败的时候才会执行这些
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                // 
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        // 是否需要扩容
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            // 数组的节点数量大于等于阈值 sizeCtl，并且数组不为空，数组长度小于最大容量则扩容
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                // 根据数组长度得到一个标识
                int rs = resizeStamp(n);
                // sc < 0 表示其他线程已经在扩容，尝试帮助扩容
                if (sc < 0) {
                    // 判断扩容是否结束，如果结束则跳出循环
                    // (sc >>> RESIZE_STAMP_SHIFT) != rs 代表 sizeCtl 发送了变化
                    // sc == rs + 1 代表扩容结束，不再有线程进行扩容
                    // sc == rs + MAX_RESIZERS 协助线程数已经达到最大
                    // (nt = nextTable) == null 代表扩容结束
                    // transferIndex <= 0 代表转移状态发生了变化
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    // 其他线程正在扩容，sc + 1 表示多了一个线程在帮助扩容
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    // 仅当前线程在扩容
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }


    /**
     * 协助扩容
     * Helps transfer if a resize is in progress.
     */
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        // 节点数组不为空，而且 f 节点的类型是 ForwardingNode 表明数组正在扩容
        // (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null 表示迁移完成了，详见transfer()
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            // 表示在扩容
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                // sc >>> RESIZE_STAMP_SHIFT) != rs 说明扩容完毕或者有其它协助扩容者
                // sc == rs + 1 表示只剩下最后一个扩容线程了，其它都扩容完毕了
                // transferIndex <= 0 扩容结束了
                // sc == rs + MAX_RESIZERS 到达最大值
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                // CAS 更新 sizeCtl=sizeCtl+1,表示新增加一个线程辅助扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }

    /**
     * Tries to presize table to accommodate the given number of elements.
     *
     * @param size number of elements (doesn't need to be perfectly accurate)
     */
    private final void tryPresize(int size) {
        // 如果给定的容量>=MAXIMUM_CAPACITY的一半，直接扩容到允许的最大值，否则调用函数计算合适的容量值
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        // 如果 sizeCtl 小于0，说明数组还没有被初始化
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                // CAS将SIZECTL状态置为-1，表示正在进行初始化
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            // 扩容值不大于原阀值，或现有容量>=最值，什么都不用做了
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    Node<K,V>[] nt;
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
    }

```


### 扩容

```

    /**
     * Moves and/or copies the nodes in each bin to new table. See
     * above for explanation.
     */
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        // 计算每个线程需要负责的迁移数量。
        // 如果 cpu 数量大于 1，迁移数量为 n >>> 3(也就是除以8) / cpu 个数，否则就数组长度。
        // 如果小于 MIN_TRANSFER_STRIDE 就直接设置线程需要负责的迁移数量为MIN_TRANSFER_STRIDE
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        // 初始化新数组
        if (nextTab == null) {            // initiating
            try {
                // 新数组的容量为原数组的两倍
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        // 创建一个 ForwardingNode 节点，用于标记
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        //并发扩容的关键属性 如果等于true 说明这个节点已经处理过
        boolean advance = true;
        // 完成状态,如果为true,就结束方法
        boolean finishing = false; // to ensure sweep before committing nextTab
        // 通过for自循环处理每个槽位中的链表元素，默认advace为真，
        // 通过CAS设置transferIndex属性值，并初始化i和bound值，i指当前处理的槽位序号，bound指需要处理的槽位边界，先处理槽位15的节点；
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                // transferIndex<=0表示已经没有需要迁移的hash桶，将i置为-1，线程准备退出
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                // CAS 操作，为当前线程分配数组索引区间
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            // i < 0 ,表示数据迁移已经完成
            // i >= n 和 i + n >= nextn 表示最后一个线程也执行完成了,扩容完成了
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 如果 finishing=true，说明扩容所有工作完成，然后跳出循环
                if (finishing) {
                    // 删除成员变量，方便GC
                    nextTable = null;
                    // 更新table
                    table = nextTab;
                    // 更新阈值
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // 表示一个线程退出扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 说明还有其他线程正在扩容
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 当前线程为最后一个线程,负责再检查一个整个队列
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            /  第一个线程获取i处的数据为null
            else if ((f = tabAt(tab, i)) == null)
                // 设置第i节点为 fwd 节点
                advance = casTabAt(tab, i, null, fwd);
            // 如果当前节点为 MOVED,说明已经处理过了,直接跳过
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                // 节点不为空,锁住i位置的头结点
                synchronized (f) {
                    // 再次核对，防止其他线程对该hash值进行修改
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        // 说明该位置存放的是普通节点，以下的操作类似 HashMap，节点的hash和数组长度与运算
                        // 如果为0则直接将原先的索引位置插入到新数组，否则重新计算数组索引位置并插入
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            // CAS 操作保证节点插入是原子性的
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        // 节点是红黑树结构
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }

```





## 参考

1. <a href="https://blog.csdn.net/ThinkWon/article/details/102506447" target="_blank">并发容器之ConcurrentHashMap详解(JDK1.8版本)与源码分析</a>
2. <a href="https://www.jianshu.com/p/9484064b308e" target="_blank">图解 Java8 ConcurrentHashMap 常用方法源码</a>
3. <a href="https://www.cnblogs.com/hello-shf/p/12183263.html" target="_blank">ConcurrentHashMap源码解析（1.8）</a>
4. <a href="https://www.jianshu.com/p/4aaf1a44ac82" target="_blank">ConcurrentHashMap源码解析(JDK1.8)</a>

