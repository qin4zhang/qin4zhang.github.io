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

## 分析
ConcurrentHashMap在Java8中的存储结构与HashMap是一样的，也是数组+链表+红黑树。与HashMap很多类似的情况我们就不做太多的分析，就看看是怎么做到保证线程安全的吧。

### 查询

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
        // 这里用volatile修饰值，是因为多线程修改的情况下。保证值的可见性，数据一致。
        volatile V val;
        // 同样，保证Node被修改，其他线程能保证其可见性。
        volatile Node<K,V> next;

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }
    }


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
        // 
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }

```








