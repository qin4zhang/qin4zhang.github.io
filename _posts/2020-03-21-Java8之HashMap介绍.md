---
layout:     post
title:      Java8之HashMap介绍
subtitle:   HashMap
date:       2020-03-21
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java8
    - HashMap
    - Map

---
# 注意
> 想法及时记录，实现可以待做。

## Map

Map常用有哪些呢？如图所示：

![Map]({{site.url}}/img/java/Java8-Map-Diagram.png)

## 简介
HashMap是Java中用于映射(键值对)处理的数据类型。本文主要基于Java8来做介绍，底层的数据结构由数组与链表和红黑树组成。

HashMap允许键和值为null，如果键为null，此时的hash码是0，这点要注意下。作为键值对映射的数据结构，只管映射关系，没有处理顺序，专注于映射的处理。

它的复杂度一般来说是常数级，也就是O(1)，但是也不绝对，如果有hash碰撞了，有可能退化为O(n)或者是O(log n)。

## 分析

本文主要介绍HashMap的查询和插入操作，扩容和hash冲突引起的链表法，以及红黑树优化措施，红黑树的介绍不在本篇。

### 构造函数
有四种构造函数，可以初始化两个主要参数，也即initialCapacity：初始大小，默认为16；loadFactor: 负载因子，默认为0.75。

```
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        // 这里注意下阈值的计算
        this.threshold = tableSizeFor(initialCapacity);
    }

    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and the default load factor (0.75).
     *
     * @param  initialCapacity the initial capacity.
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    /**
     * Constructs a new <tt>HashMap</tt> with the same mappings as the
     * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
     * default load factor (0.75) and an initial capacity sufficient to
     * hold the mappings in the specified <tt>Map</tt>.
     *
     * @param   m the map whose mappings are to be placed in this map
     * @throws  NullPointerException if the specified map is null
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

### tableSizeFor作用

从构造函数中可以看到，有这么一个方法来计算阈值。

```
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

可以看到，设置的容量经过计算后才能得到阈值，规则如上所示，大量的位运算，那么究竟想要得到什么结果呢？

不妨来归纳下：

情况1：cap为2的次幂，输入为4，那么输出呢？

a: n = 4-1 = 3;

b: n |= n >>> 1 = n | (n >>> 1) = 0011 | 0001 = 0011

c: n |= n >>> 2 = n | (n >>> 2) = 0011 | 0000 = 0011

d: n |= n >>> 4 = n | (n >>> 4) = 0011 | 0000 = 0011

...           8

...           16

位运算后n=0011，也即3.经过三目运算符后得到结果为 3+1=4，也即2的2次幂。

情况2：cap为5

a: n = 5-1 = 4;

b: n |= n >>> 1 = n | (n >>> 1) = 0100 | 0010 = 0110

c: n |= n >>> 2 = n | (n >>> 2) = 0110 | 0001 = 0111

d: n |= n >>> 4 = n | (n >>> 4) = 0111 | 0000 = 0111

...           8

...           16

位运算后n=0111，也即7.经过三目运算符后得到结果为 7+1=8，也即2的3次幂。

寻找对于cap大于等于最近的2的次幂的一个数，即为阈值。

### 查询

```

    /**
     * HashMap的基本的存储Node结构
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        // 不可变的hash值，一旦Node初始化，该值不在允许改变。
        final int hash;
        // 不可变键
        final K key;
        // 值，没有那么多限制，毕竟允许修改。
        V value;
        // next引用，非常有用，可以组成单向链表。
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }

    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    // 计算键的hash值，hash规则为int类型的数据，高16位和低16位异或运算求值。
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }


    // 根据键的hash值和键来寻找匹配的数据
    final Node<K,V> getNode(int hash, Object key) {
        // tab为存储数据的数组，first为hash值相同的第一个匹配，e为下一个值，n为数组长度,k为键
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        // 数组不为null，并且数组长度大于0，并且在数组中的第一个值不为null，说明有可能存在，这里是通过数组的下标取值，查找是O(1)。
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 如果第一个node满足hash值相等，并且键相等，那么第一个节点就是要找的，这种是不存在hash冲突的大多数情况。复杂度为O(1)。
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            // 下一个node不为null
            if ((e = first.next) != null) {
                // 如果第一个node是红黑树，就按照树的方式去寻找，树的复杂度为O(log n)
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 否则就按照链表的方式循环查找
                do {
                    // 判断条件为：node的hash值匹配并且键相等，否则一直遍历完链表的每一个数据。这里的最糟糕的情况，就是O(n)。
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        // 不存在hash和key满足的键值对
        return null;
    }

```

### 插入

插入操作涉及逻辑主要有：
1. 当存储数组为null或者长度为0，通过扩容初始化数组
2. 查找要插入的键值对是否存在，根据条件决定是否更新旧值
3. 如果不存在，将node插入到链表中，根据链表长度决定是否转红黑树
4. 根据最终的map大小与阈值比较，决定是否进行扩容。

```
    // 常用的插入键值对的方式
    public V put(K key, V value) {
        // 键的hash值
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * 实际执行插入的逻辑
     * Implements Map.put and related methods.
     *
     * @param hash hash for key 键的hash值，规则见上文。
     * @param key the key 键
     * @param value the value to put 键对应的值
     * @param onlyIfAbsent if true, don't change existing value 如果为true，表示不修改已存在的值，仅修改不存在的值。
     * @param evict if false, the table is in creation mode. 如果为false，表示处理创建模式。
     * @return previous value, or null if none 返回前一个值，如果没有则返回null
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 存储数组为空，或者存储数组长度为0
        if ((tab = table) == null || (n = tab.length) == 0)
            // 进行扩容，然后得到此时的数组长度。具体的扩容逻辑，暂时不管，后面再看。
            n = (tab = resize()).length;
        // 根据下标获取数组的具体node数据，如果为null，说明当前没有值也就是没有hash冲突，可以新增。
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            // 以下的情况都是hash冲突的处理了。
            Node<K,V> e; K k;
            // 冲突的这个node的hash值与插入的hash值相等，并且key相等，说明是同一个key。
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 冲突的这个node，key也不一样，说明是有新的key引起的hash冲突，如果已经是红黑树，按照红黑树的方式新增
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
            // 冲突的这个node，key也不一样，说明是有新的key引起的hash冲突，不是红黑树，那就是链表结构
                // 循环遍历链表
                for (int binCount = 0; ; ++binCount) {
                    // 下一个节点如果为空，则往链表尾新增数据。
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 插入数据的时候，达到红黑树化的条件，就可以进行构造红黑树。链表的长度达到8，就开始转树。
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        // 下一个节点都为null，本次插入可以结束。
                        break;
                    }
                    // 下一个节点不为null，说明有值。判断与插入的hash与key是否一致。如果一致，说明是已存在的节点，跳出循环。
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    // 更新指向node的下一个节点
                    p = e;
                }
            }
            // 要插入的node是否存在于map中,如果已经存在于map中，那么本次插入操作不产生结构的影响，最多只改变该node的值。
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                // 这里判断是不是更新值，也就是键对应的值为null，可以更新值，或者设置的策略允许修改值，也可以。
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                // 回调的方法，HashMap空实现，LinkedHashMap有大作用。
                afterNodeAccess(e);
                return oldValue;
            }
        }
        // 这个修改次数，用于迭代器使用，如果前后不等，引起的异常就是 ConcurrentModificationException。这个就是fast fail的处理。
        ++modCount;
        // 除了第一次如果存储数组为空需要扩容，后面的插入数据，如果达到阈值，也要进行扩容。
        if (++size > threshold)
            resize();
        // 回调的方法，HashMap空实现，LinkedHashMap有大作用。
        afterNodeInsertion(evict);
        return null;
    }
```

### 扩容

数组的大小，都是一开始就确定好的，如果大小不合适，就需要进行扩容，这个时候就要将老的数据迁移到新的数组中。

迁移至新的数据中，下标计算规则也很简单，要么是原有的下标不变，要么是原有的下标加上原有的容量作为新的下标，分布很均匀。

```
    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     * 元素在新的数组中，要么index不变，要么偏移为2的次幂，也就是 index + 2 ^ x = 新的index
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        // 局部变量，老的数组
        Node<K,V>[] oldTab = table;
        // 局部变量，老的容量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        // 老的阈值
        int oldThr = threshold;
        // 新的容量与新的阈值，初始值为0
        int newCap, newThr = 0;
        // 老的容量大于0的时候，已经过了初始化阶段。
        if (oldCap > 0) {
            // 超过最大的容量2 ^ 30，阈值直接给最大整型值(2 ^ 31 - 1)，不在扩容。
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 按旧容量和阈值的2倍计算新容量和阈值的大小
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        // 老的阈值大于0的时候，此时是构造函数带参数的情况
        else if (oldThr > 0) // initial capacity was placed in threshold
            // 新的容量为老的阈值
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 都是默认的初始容量与初始计算得到的阈值，此时是无参构造函数。
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 如果新的阈值为0，也就是没有被赋值，但是新的容量一定是有值的。
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        // 新容量的数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            // 老的数组不为null，遍历老的数组，将键值对插入到新的数组中，做数据迁移。
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                // 取出老数组的数据到临时变量中
                if ((e = oldTab[j]) != null) {
                    // 老数组的数据置为null，便于GC
                    oldTab[j] = null;
                    // 这个数据的下一个数据为null，则不会有hash冲突，直接插入到新的数组中。
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    // 下一个数据不为null，说明存在hash冲突，链表法，存储为链表或者红黑树
                    else if (e instanceof TreeNode)
                        // 如果是红黑树，则需要对红黑树进行拆分
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 链表的存储，保持原有顺序不变
                        // 低位的头结点和尾节点
                        Node<K,V> loHead = null, loTail = null;
                        // 高位的头结点和尾节点
                        Node<K,V> hiHead = null, hiTail = null;
                        // 临时变量，下一个节点
                        Node<K,V> next;
                        // 该节点是链表，循环遍历处理
                        do {
                            next = e.next;
                            // 这里的处理很巧妙，单独分析。将原本的链表节点，进行分组处理。
                            // index不变
                            if ((e.hash & oldCap) == 0) {
                                // 低位尾节点为null，说明当前节点未初始化，将节点赋值给头结点。
                                if (loTail == null)
                                    loHead = e;
                                // 尾节点有值，那么往尾节点后面继续加
                                else
                                    loTail.next = e;
                                // 将该节点赋值给尾节点。
                                loTail = e;
                            }
                            // index = index + oldCap
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 循环遍历完节点的链表存储
                        // 低位尾节点不为null，那么将低位的头结点的node，存储到新的数组中，下标不变。
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // 高位尾节点不为null，那么将高位的头结点的node，存储到新的数组中，下标为 原有下标+老的容量
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

对于hash冲突引起的链表法，扩容的时候，是怎么处理成两条链的呢？这里来分析下。

根据最后给新的数组赋值的操作可以看出：数据的位置要么是原位置，要么是原位置加原数组长度。

我们知道数组的长度是2的次幂，不管是默认的初始值即16，或者是手动设置的cap值，最后经过上文构造函数介绍的处理，阈值还是2的次幂，最后初始化的时候，阈值也赋值给了容量，最后数组的长度始终还是2的次幂。

这是个前提，对于2的次幂，我们看到很多次计算数组的下标的操作为 (n - 1) & hash，这个n是2的次幂，与（hash % n）效果一致。也就是 hash % n = (n - 1 ) & hash。位运算更快，取模计算数组下标很好理解。

但是对于 hash & n 这个操作什么效果呢？

由于n是2的次幂，那么除了高位有1，其余位全部为0，而hash值是很随机的，此时进行与运算，hash值与高位要么为0，要么为1，没有其他选择，也就将hash值分为两组了。

分为两组解释清楚了，那为什么下标要么不变，要么为原下标加数组长度呢？

扩容前，n-1的结果是高位为0，其余为为1，这样 hash & (n - 1) = index算的扩容前的下标。

扩容后n的高位左移一位，n-1的结果是高位0也左移一位，于是与hash相对应的原先的高位变为1了。

这样hash不变的情况下，hash原先的自己对应的位为0，则因为 (新的容量对应位)1 & (hash对应位)0 = 0，所以与运算后结果不变，还是原来的index；

如果hash原先自己对应的位是1，则因为 (新的容量对应位)1 & (hash对应位)1 = 1 所以与运算后，扩容前的高位1得以保留，也就是扩容前的容量算作扩容后的下标一部分，容量的剩下位都是1，与扩前后无影响。

这也就是，老的容量的高位与运算后，为0，就是原有下标，为1，就要加下原有容量作为偏移量。


### 链表转红黑树
插入新的键值对的时候，如果hash冲突导致的链表长度达到了8，就要进行树化，因为时间复杂度由 O(n) -> O(log n)。注意：8这个是根据泊松分布得到的概率非常小的冲突的值，可以看注释。

注意：在插入新的键值对的过程中，链表转为红黑树需要满足两个条件：
1. 链表的长度要大于等于 8(TREEIFY_THRESHOLD)
2. 存储数据的长度要大于等于64(MIN_TREEIFY_CAPACITY)

```

    static final int TREEIFY_THRESHOLD = 8;
    
    /**
     * 当桶数组容量小于该值时，优先进行扩容，而不是树化
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

    /**
     * Entry for Tree bins. Extends LinkedHashMap.Entry (which in turn
     * extends Node) so can be used as extension of either regular or
     * linked node.
     * 红黑树的节点定义，继承了LinkedHashMap.Entry，那也继承了HashMap.Node，所以也保留了原有Node的属性。
     */
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
        ...
    }

    /**
     * HashMap.Node subclass for normal LinkedHashMap entries.
     */
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }

    /**
     * Replaces all linked nodes in bin at index for given hash unless
     * table is too small, in which case resizes instead.
     */
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        // 存储数组如果为空，或者数组的长度小于最小树化容量64，说明数量还比较少，优先扩容处理，不会树化。
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        // 存储数据的长度已经不小于64，则进行链表转红黑树的构造。
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            // hd -> 头节点，tl -> 尾节点
            TreeNode<K,V> hd = null, tl = null;
            do {
                // 将普通的节点转为树节点
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                // 真正的转为红黑树
                hd.treeify(tab);
        }
    }

    // For treeifyBin
    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        return new TreeNode<>(p.hash, p.key, p.value, next);
    }
```

### 红黑树拆分

扩容后，红黑树的节点也需要重新处理，这就涉及到红黑树的拆分了。

将原有的红黑树拆分成两条红黑树节点的链表，分别看每条链表是不是需要转为普通的节点链表，或者保持红黑树不变。

```
        // 红黑树转链表阈值
        static final int UNTREEIFY_THRESHOLD = 6;

        /**
         * Splits nodes in a tree bin into lower and upper tree bins,
         * or untreeifies if now too small. Called only from resize;
         * see above discussion about split bits and indices.
         *
         * @param map the map
         * @param tab the table for recording bin heads
         * @param index the index of the table being split
         * @param bit the bit of hash to split on
         */
        final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            // 红黑树的TreeNode保留了普通的Node的next引用，所以还是可以按链表的方式遍历树
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                // 这里跟链表扩容的时候很像，分组处理
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

            if (loHead != null) {
                // 低位数据，长度小于等于6，则转为链表，下标不变。
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    // 仍旧是红黑树
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                // 高位数据，长度小于等于6，则转为链表，下标还是原有下标加bit的偏移量。
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    // 仍旧是红黑树
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }

```



### 红黑树转链表

扩容的时候，如果红黑树拆分的红黑树节点的链表长度符合要求了，就会转为普通节点的链表。

```
    /**
     * Returns a list of non-TreeNodes replacing those linked from
     * this node.
     */
    final Node<K,V> untreeify(HashMap<K,V> map) {
        Node<K,V> hd = null, tl = null;
        // 遍历TreeNode的链表
        for (Node<K,V> q = this; q != null; q = q.next) {
            // 将红黑树节点转为普通的Node节点
            Node<K,V> p = map.replacementNode(q, null);
            if (tl == null)
                hd = p;
            else
                tl.next = p;
            tl = p;
        }
        return hd;
    }

    // For conversion from TreeNodes to plain nodes
    Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
        return new Node<>(p.hash, p.key, p.value, next);
    }

``` 

## 参考

1. <a href="https://segmentfault.com/a/1190000012926722" target="_blank">HashMap 源码详细分析(JDK1.8)</a>
2. <a href="https://yanglukuan.github.io/2017/08/31/java/HashMap%E8%AF%A6%E8%A7%A3/" target="_blank">Java集合框架之HashMap详解</a>
3. <a href="https://tech.meituan.com/2016/06/24/java-hashmap.html" target="_blank">Java 8系列之重新认识HashMap</a>

