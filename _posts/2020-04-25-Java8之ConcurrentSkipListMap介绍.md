---
layout:     post
title:      Java8之ConcurrentSkipListMap介绍
subtitle:   ConcurrentSkipListMap
date:       2020-04-25
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java8
    - ConcurrentSkipListMap
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

如果不熟悉HashMap，可以看<a href="https://qin4zhang.github.io/2020/03/21/Java8%E4%B9%8BHashMap%E4%BB%8B%E7%BB%8D/" target="_blank">Java8之HashMap介绍</a>

如果我们需要多线程环境下，还要保证有序数据存储，那么CHM也无法满足，此时，ConcurrentSkipListMap这种结构便有机会使用。

## 分析
ConcurrentSkipListMap是一个并发安全的，基于SkipList实现的有序存储的Map。

跳表是一个链表结构，写入和读取的时间复杂度在O(log n)，利用空间换时间，增加多层索引，提升查找和写入效率。

跳表具有以下几个的特点：

1. 最底层包含所有节点的一个有序链表
2. 每一层都是有序的链表
3. 每个节点都有两个指针，一个指向下一节点，一个指向下一层节点
4. 如果一个元素出现在level(x)中，那么它肯定出现在x一下的所有level中

### 组成介绍

```
    /**
     * 用于标识基本级别标头的特殊值
     * Special value used to identify base-level header
     */
    private static final Object BASE_HEADER = new Object();

    /**
     * 跳表的最高头索引
     * The topmost head index of the skiplist.
     */
    private transient volatile HeadIndex<K,V> head;

    /**
     * 比较器用于维护此映射中的顺序，如果使用自然顺序，则为null。 （非私有可简化嵌套类的访问。）
     * The comparator used to maintain order in this map, or null if
     * using natural ordering.  (Non-private to simplify access in
     * nested classes.)
     * @serial
     */
    final Comparator<? super K> comparator;

    /** Lazily initialized key set */
    private transient KeySet<K> keySet;
    /** Lazily initialized entry set */
    private transient EntrySet<K,V> entrySet;
    /** Lazily initialized values collection */
    private transient Values<V> values;
    /** Lazily initialized descending key set */
    private transient ConcurrentNavigableMap<K,V> descendingMap;


```

### 关键的内部类

```

    /**
     * 头索引，扩展索引，多了一个字段，记录索引所在的层级
     * Nodes heading each level keep track of their level.
     */
    static final class HeadIndex<K,V> extends Index<K,V> {
        final int level;
        HeadIndex(Node<K,V> node, Index<K,V> down, Index<K,V> right, int level) {
            super(node, down, right);
            this.level = level;
        }
    }

    /**
     * 数据节点，存储数据，单链表结构
     * Nodes hold keys and values, and are singly linked in sorted
     * order, possibly with some intervening marker nodes. The list is
     * headed by a dummy node accessible as head.node. The value field
     * is declared only as Object because it takes special non-V
     * values for marker and header nodes.
     */
    static final class Node<K,V> {
        // 节点的key是不可变的
        final K key;
        // 使用 volatile 修饰，保障 happens-before 语义以及线程间可见性。
        volatile Object value;
        volatile Node<K,V> next;
    }

    /**
     * 索引，存储着对应的Node值、向右向下的索引
     * Index nodes represent the levels of the skip list.  Note that
     * even though both Nodes and Indexes have forward-pointing
     * fields, they have different types and are handled in different
     * ways, that can't nicely be captured by placing field in a
     * shared abstract class.
     */
    static class Index<K,V> {
        final Node<K,V> node;
        final Index<K,V> down;
        volatile Index<K,V> right;

        /**
         * Creates index node with given values.
         */
        Index(Node<K,V> node, Index<K,V> down, Index<K,V> right) {
            this.node = node;
            this.down = down;
            this.right = right;
        }
    }

```

### 查找

```
    /**
     * 获取key对应的值，如果没有找到，就返回null
     * Gets value for key. Almost the same as findNode, but returns
     * the found value (to avoid retries during re-reads)
     *
     * @param key the key
     * @return the value, or null if absent
     */
    private V doGet(Object key) {
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            // b是key的前继节点,n是b的下一个节点，key对应的就是在 b和n之间
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                // 说明是链表的最后一个节点，为null，那自然没有找到了，直接挑出最外层循环
                if (n == null)
                    break outer;
                // f是n的下一个节点
                Node<K,V> f = n.next;
                // 两次读取的值不一致，说明有变化，进行查找重试
                if (n != b.next)                // inconsistent read
                    break;
                // n的值如果是null，则删除，然后进行查找的重试
                if ((v = n.value) == null) {    // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                // 前继节点被删除了，重试
                if (b.value == null || v == n)  // b is deleted
                    break;
                // 找到了key对应的值，返回
                if ((c = cpr(cmp, key, n.key)) == 0) {
                    @SuppressWarnings("unchecked") V vv = (V)v;
                    return vv;
                }
                // 没有找到key，跳出外层循环，
                if (c < 0)
                    break outer;
                // key还要往下一个范围继续找
                b = n;
                n = f;
            }
        }
        return null;
    }

    /**
     * 查找key的前继节点，从head节点开始遍历，找到小于key的最大的一个索引节点指向的节点
     * Returns a base-level node with key strictly less than given key,
     * or the base-level header if there is no such node.  Also
     * unlinks indexes to deleted nodes found along the way.  Callers
     * rely on this side-effect of clearing indices to deleted nodes.
     * @param key the key
     * @return a predecessor of key
     */
    private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
        if (key == null)
            throw new NullPointerException(); // don't postpone errors
        for (;;) {
            // q是head节点，r是q的右节点
            for (Index<K,V> q = head, r = q.right, d;;) {
                // 右节点不为null
                if (r != null) {
                    Node<K,V> n = r.node;
                    K k = n.key;
                    // r这个节点未nul，则会被删除
                    if (n.value == null) {
                        // 从q中删除，cas失败则重试
                        if (!q.unlink(r))
                            break;           // restart
                        // 赋值新的right节点，继续循环
                        r = q.right;         // reread r
                        continue;
                    }
                    // 比较器比较 key > k，则继续向右查找
                    if (cpr(cmp, key, k) > 0) {
                        q = r;
                        r = r.right;
                        continue;
                    }
                }
                // 右节点为null，则down下去，如果down下去节点未null，则返回q的节点
                if ((d = q.down) == null)
                    return q.node;
                // down下去的节点不为null，继续下一个迭代，更新down和right方向的节点
                q = d;
                r = d.right;
            }
        }
    }


```

### 插入


```

    /**
     * Main insertion method.  Adds element if not present, or
     * replaces value if present and onlyIfAbsent is false.
     * @param key the key
     * @param value the value that must be associated with key
     * @param onlyIfAbsent if should not insert if already present
     * @return the old value, or null if newly inserted
     */
    private V doPut(K key, V value, boolean onlyIfAbsent) {
        Node<K,V> z;             // added node
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        // 外层循环
        outer: for (;;) {
            // 根据 key，找到待插入的位置
            // b 叫做前驱节点，将来作为新加入结点的前驱节点
            // n 叫做后继结点，将来作为新加入结点的后继结点
            // 也就是说，新节点将插入在 b 和 n 之间
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                // 如果 n 为 null，那么说明 b 是链表的最尾端的结点，这种情况比较简单，直接构建新节点插入即可
                // n不为null，说明不是链表的最后一个节点
                if (n != null) {
                    Object v; int c;
                    // f是n的下一个节点
                    Node<K,V> f = n.next;
                    // 如果 n 不再是 b 的后继结点了，说明有其他线程向 b 后面添加了新元素
                    // 那么我们直接退出内循环，重新计算新节点将要插入的位置
                    if (n != b.next)               // inconsistent read
                        break;
                    // 如果n的值为null，说明n已经删除，这里则帮助删除，重试
                    if ((v = n.value) == null) {   // n is deleted
                        n.helpDelete(b, f);
                        break;
                    }
                    // 前驱节点b删除，重试
                    if (b.value == null || v == n) // b is deleted
                        break;
                    // 如果 插入的key比 n的key还要大，说明n范围不够，需要继续往后找，将n作为前驱节点，f作为n，插入的key继续在此范围内寻找。
                    if ((c = cpr(cmp, key, n.key)) > 0) {
                        b = n;
                        n = f;
                        continue;
                    }
                    // n的key就是本次要插入的key
                    if (c == 0) {
                        // 如果要覆盖n的值，则cas赋值，成功则返回插入的值，否则因为cas竞争失败，则重试
                        if (onlyIfAbsent || n.casValue(v, value)) {
                            @SuppressWarnings("unchecked") V vv = (V)v;
                            return vv;
                        }
                        break; // restart if lost race to replace value
                    }
                    // 
                    // else c < 0; fall through
                }
                // 没有找到，则本次新插入的节点
                z = new Node<K,V>(key, value, n);
                // n是链表的最后一个节点，此时cas往链表后追加，cas竞争失败，则重试
                if (!b.casNext(n, z))
                    break;         // restart if lost race to append to b
                // 全部走完，跳出外层的for循环，不在重试
                break outer;
            }
        }
        // 获取一个线程无关的随机数，占四个字节，32 个比特位
        int rnd = ThreadLocalRandom.nextSecondarySeed();
        // 和 1000 0000 0000 0000 0000 0000 0000 0001 与
        // 如果等于 0，说明这个随机数最高位和最低位都为 0，这种概率很大
        // 如果不等于 0，那么将仅仅把新节点插入到最底层的链表中即可，不会往上层递归
        if ((rnd & 0x80000001) == 0) { // test highest and lowest bits
            int level = 1, max;
            // 用低位连续为 1 的个数作为 level 的值，也是一种概率策略
            while (((rnd >>>= 1) & 1) != 0)
                ++level;
            // 上面这段代码是获取 level 的, 我们这里只需要知道获取 level 就可以 (50%的几率返回0，25%的几率返回1，12.5%的几率返回2...最大返回31。)
            Index<K,V> idx = null;
            HeadIndex<K,V> h = head;
            // 如果概率算得的 level 在当前跳表 level 范围内
            // 构建一个从 1 到 level 的纵列 index 结点引用
            // 初始化 max 的值, 若 level <= max , 创建一个通过down属性构成的Index链表
            if (level <= (max = h.level)) {
                for (int i = 1; i <= level; ++i)
                    // idx是作为新节点的down节点的
                    idx = new Index<K,V>(z, idx, null);
            }
            // 若 level > max 则只增加一层 index 索引层
            else { // try to grow by one level
                level = max + 1; // hold in array and later pick the one to use
                // 创建一个Index数组
                @SuppressWarnings("unchecked")Index<K,V>[] idxs =
                    (Index<K,V>[])new Index<?,?>[level+1];
                // 初始化数组元素
                for (int i = 1; i <= level; ++i)
                    idxs[i] = idx = new Index<K,V>(z, idx, null);
                for (;;) {
                    h = head;
                    // 获取老的 level 层
                    int oldLevel = h.level;
                    // 初始状态下level肯定大于oldLevel
                    if (level <= oldLevel) // lost race to add level
                        break;
                    HeadIndex<K,V> newh = h;
                    Node<K,V> oldbase = h.node;
                    // 创建新层级对应的head节点，down节点指向原来的head，right节点指向新节点z对应的索引节点
                    // 通常oldLevel就比level小1，即只创建一个HeadIndex节点
                    for (int j = oldLevel+1; j <= level; ++j)
                        // idxs[j] 就是上面的 idxs中的最高层的索引
                        newh = new HeadIndex<K,V>(oldbase, newh, idxs[j], j);
                    // cas修改head，如果修改失败说明其他线程在修改head，则重新for循环读取head
                    if (casHead(h, newh)) {
                        h = newh;
                        idx = idxs[level = oldLevel];
                        break;
                    }
                }
            }
            // 如果新节点z对应的level不是0，则创建对应层级的down链表，
            // 如果层级大于head的还需要调整head的层级，创建新的head节点，新层级的head节点的right节点就指向新节点z
            // find insertion points and splice in
            splice: for (int insertionLevel = level;;) {
                int j = h.level;
                for (Index<K,V> q = h, r = q.right, t = idx;;) {
                    // 节点删除了，循环结束
                    if (q == null || t == null)
                        break splice;
                    if (r != null) {
                        Node<K,V> n = r.node;
                        // compare before deletion check avoids needing recheck
                        int c = cpr(cmp, key, n.key);
                        // r节点是无效的
                        if (n.value == null) {
                            // 将r从q的right链表中移除，移除失败重新读取q
                            if (!q.unlink(r))
                                break;
                            // 获取新的right节点
                            r = q.right;
                            continue;
                        }
                        // key大于n.key，说明前驱节点不对，需要向右继续遍历
                        if (c > 0) {
                            q = r;
                            r = r.right;
                            continue;
                        }
                    }

                    if (j == insertionLevel) {
                        // 将t插入到q的right链表中，插入到r的前面，失败则重试
                        if (!q.link(r, t))
                            break; // restart
                        // 正常不可能为null，极端并发下可能为null，因为此时已经将新节点插入到链表中了，有可能被其他某个线程给删除了
                        if (t.node.value == null) {
                            // 通过findNode将该节点移除，然后终止外层for循环
                            findNode(key);
                            break splice;
                        }
                        // 到底层节点了，终止外层for循环
                        if (--insertionLevel == 0)
                            break splice;
                    }
                    // 如果j大于level，说明未遍历到新节点所在的最高层级，小于level说明上一个层级的right节点处理好了，需要遍历下一个层级
                    if (--j >= insertionLevel && j < level)
                        t = t.down;
                    // 遍历下一个层级的节点
                    q = q.down;
                    r = q.right;
                }
            }
        }
        return null;
    }

```

### 删除

```

    /**
     * Main deletion method. Locates node, nulls value, appends a
     * deletion marker, unlinks predecessor, removes associated index
     * nodes, and possibly reduces head index level.
     *
     * Index nodes are cleared out simply by calling findPredecessor.
     * which unlinks indexes to deleted nodes found along path to key,
     * which will include the indexes to this node.  This is done
     * unconditionally. We can't check beforehand whether there are
     * index nodes because it might be the case that some or all
     * indexes hadn't been inserted yet for this node during initial
     * search for it, and we'd like to ensure lack of garbage
     * retention, so must call to be sure.
     *
     * @param key the key
     * @param value if non-null, the value that must be
     * associated with key
     * @return the node, or null if not found
     */
    final V doRemove(Object key, Object value) {
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        // 外层循环
        outer: for (;;) {
            // 查找key的前驱节点
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                if (n == null)
                    break outer;
                Node<K,V> f = n.next;
                if (n != b.next)                    // inconsistent read
                    break;
                if ((v = n.value) == null) {        // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                if (b.value == null || v == n)      // b is deleted
                    break;
                // 正常情况下，key 应该等于 n.key
                // key 大于 n.key 说明我们要找的结点可能在 n 的后面，往后递归即可
                // key 小于 n.key 说明 key 所代表的结点根本不存在
                if ((c = cpr(cmp, key, n.key)) < 0)
                    break outer;
                if (c > 0) {
                    b = n;
                    n = f;
                    continue;
                }
                if (value != null && !value.equals(v))
                    break outer;
                // 尝试将待删结点的 value 属性赋值 null，失败将退出重试
                if (!n.casValue(v, null))
                    break;
                // casValue成功，在n和f之间插入一个空节点，如果成功则将b的next节点修改成f
                if (!n.appendMarker(f) || !b.casNext(n, f))
                    findNode(key);                  // retry via findNode
                else {
                    // 修改成功，通过findPredecessor清理关联的Index节点
                    findPredecessor(key, cmp);      // clean index
                    // 判断此次删除后是否导致某一索引层没有其他节点了，并适情况删除该层索引
                    if (head.right == null)
                        tryReduceLevel();
                }
                @SuppressWarnings("unchecked") V vv = (V)v;
                return vv;
            }
        }
        return null;
    }

```





## 参考

1. <a href="https://houbb.github.io/2020/10/17/lock-09-ConcurrentSkipListMap-source-code" target="_blank">ConcurrentSkipListMap 源码解析</a>
2. <a href="https://www.jianshu.com/p/edc2fd149255" target="_blank">ConcurrentSkipListMap 源码分析 (基于Java 8)</a>
3. <a href="https://cloud.tencent.com/developer/article/1013646" target="_blank">基于跳跃表的 ConcurrentSkipListMap 内部实现（Java 8）</a>
4. <a href="https://blog.csdn.net/qq_31865983/article/details/105586590" target="_blank">Java8 ConcurrentSkipListMap与ConcurrentSkipListSet （一） 源码解析</a>

