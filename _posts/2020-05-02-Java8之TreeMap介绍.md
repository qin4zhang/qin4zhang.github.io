---
layout:     post
title:      Java8之TreeMap介绍
subtitle:   TreeMap
date:       2020-05-02
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java8
    - TreeMap
    - Map
    - Red-Black Tree

---
# 注意
> 想法及时记录，实现可以待做。

## Map

Map常用有哪些呢？如图所示：

![Map]({{site.url}}/img/java/Java8-Map-Diagram.png)

## 简介
HashMap是Java中用于映射(键值对)处理的数据类型。但是属于无序的数据结构，如果我们还要想有序的映射，怎么办呢？此时TreeMap就很合适了。

HashMap允许键和值为null，但是TreeMap不允许键为null，因为键涉及到排序使用，值可以为null。

## 分析
TreeMap的底层存储结构，使用的是红黑树，所以它的基本操作都是O(log n)的时间复杂度。它的映射排序是根据键的自然顺序进行排序，或者指定比较器进行排序。

注意，TreeMap是非线程安全的，并发情况下肯定有问题，注意控制。

红黑树是一颗自平衡二叉树，它具有以下几个的特点：

1. 每个节点只能是红色或者黑色
2. 根节点一定是黑色
3. 每个叶子节点（NIL节点，空节点）是黑色
4. 如果一个节点是红色，那么它的两个子节点一定是黑色。也就是说，一条路径上，不能出现相邻的两个红色节点。
5. 从任一节点到其每个叶子节点的所有路径都包含相同数目的黑色节点。

### 组成介绍

```

    /**
     * 这个比较器，用来维护map的顺序的，如果比较器为null，则使用key的自然顺序作为排序。
     * The comparator used to maintain order in this tree map, or
     * null if it uses the natural ordering of its keys.
     *
     * @serial
     */
    private final Comparator<? super K> comparator;

    // 根节点
    private transient Entry<K,V> root;

    /**
     * 树的节点个数
     * The number of entries in the tree
     */
    private transient int size = 0;

    /**
     * 修改树结构的次数
     * The number of structural modifications to the tree.
     */
    private transient int modCount = 0;



```

### 关键的内部类

```

    /**
     * 无参构造器。使用 key 的自然顺序排列（key 要实现 Comparable 接口）
     */
    public TreeMap() {
        comparator = null;
    }
    
    /**
     * 使用指定的 Comparator（比较器）构造一个空的 TreeMap
     */
    public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
    }
    
    /**
     * 使用给定的 Map 构造一个 TreeMap
     */
    public TreeMap(Map<? extends K, ? extends V> m) {
        comparator = null;
        putAll(m);
    }
    
    /**
     * 使用给定的 SortedMap 构造一个 TreeMap
     *（使用 SortedMap 的顺序）
     */
    public TreeMap(SortedMap<K, ? extends V> m) {
        comparator = m.comparator();
        try {
            buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
        } catch (java.io.IOException cannotHappen) {
        } catch (ClassNotFoundException cannotHappen) {
        }
    }

    /**
     * Node in the Tree.  Doubles as a means to pass key-value pairs back to
     * user (see Map.Entry).
     */
    static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        // 指向左子树
        Entry<K,V> left;
        // 指向右子树
        Entry<K,V> right;
        // 指向父节点
        Entry<K,V> parent;
        // 每个节点的颜色，红色或者黑色
        boolean color = BLACK;
    }

```

### 查找

```

    /**
     * 查找key对应的节点，否则返回null
     * Returns this map's entry for the given key, or {@code null} if the map
     * does not contain an entry for the key.
     *
     * @return this map's entry for the given key, or {@code null} if the map
     *         does not contain an entry for the key
     * @throws ClassCastException if the specified key cannot be compared
     *         with the keys currently in the map
     * @throws NullPointerException if the specified key is null
     *         and this map uses natural ordering, or its comparator
     *         does not permit null keys
     */
    final Entry<K,V> getEntry(Object key) {
        // Offload comparator-based version for sake of performance
        // 比较器不为null，则按照传入的比较器查询数据。仔细看，这里的接口是Comparator
        if (comparator != null)
            return getEntryUsingComparator(key);
        // key不能为null
        if (key == null)
            throw new NullPointerException();
        // 如果没有自定义的比较器，那么就用key自身实现的比较方法。仔细看，这里的接口是Comparable，key要实现这个接口。
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        Entry<K,V> p = root;
        // 根节点不为null，循环查找子树，根据key做比较后就知道在树的哪一边
        // key比节点要小，则向左子树找；比节点大，则向右子树找；刚好找到，就返回。树都遍历完了，还没找到，就返回null。很常规的二叉树查找。
        while (p != null) {
            int cmp = k.compareTo(p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
        return null;
    }

    /**
     * 使用自定义的比较器查找数据
     * Version of getEntry using comparator. Split off from getEntry
     * for performance. (This is not worth doing for most methods,
     * that are less dependent on comparator performance, but is
     * worthwhile here.)
     */
    final Entry<K,V> getEntryUsingComparator(Object key) {
        @SuppressWarnings("unchecked")
            K k = (K) key;
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            Entry<K,V> p = root;
            // 二叉树的查找没啥特别的，这里只是换了个比较器来比较key之间的大小，其他都还是树的查找。
            while (p != null) {
                int cmp = cpr.compare(k, p.key);
                if (cmp < 0)
                    p = p.left;
                else if (cmp > 0)
                    p = p.right;
                else
                    return p;
            }
        }
        return null;
    }

```

### 插入

```

    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     *
     * @return the previous value associated with {@code key}, or
     *         {@code null} if there was no mapping for {@code key}.
     *         (A {@code null} return can also indicate that the map
     *         previously associated {@code null} with {@code key}.)
     * @throws ClassCastException if the specified key cannot be compared
     *         with the keys currently in the map
     * @throws NullPointerException if the specified key is null
     *         and this map uses natural ordering, or its comparator
     *         does not permit null keys
     */
    public V put(K key, V value) {
        Entry<K,V> t = root;
        // 根节点为null，则初始化
        if (t == null) {
            compare(key, key); // type (and possibly null) check

            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        // 自定义比较器不为null，用这个确定位置
        if (cpr != null) {
            // 根据自定义的比较器，找到可以插入节点的位置的父节点
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        else {
            // 这里是根据key实现的比较器，确定插入节点的位置的父节点
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        // 初始化新节点，然后根据上面找到的节点位置的父节点，将新节点依据判断大小连接到父节点的左子树或者右子树。
        Entry<K,V> e = new Entry<>(key, value, parent);
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
        // 上面只是一般的二叉树的插入，但是红黑树是自平衡的，所以接下来就要调整节点，保证是平衡二叉树
        fixAfterInsertion(e);
        size++;
        modCount++;
        return null;
    }

```





## 参考

1. <a href="https://juejin.cn/post/6844903921912119310" target="_blank">TreeMap源码分析（基于jdk1.8）</a>
1. <a href="https://www.cnblogs.com/nananana/p/10426377.html" target="_blank">TreeMap实现原理及源码分析之JDK8</a>

