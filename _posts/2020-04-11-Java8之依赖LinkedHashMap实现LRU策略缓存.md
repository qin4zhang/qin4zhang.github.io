---
layout:     post
title:      Java8之依赖LinkedHashMap实现LRU策略缓存
subtitle:   HashMap
date:       2020-04-11
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java8
    - LinkedHashMap
    - HashMap
    - Map

---
# 注意
> 想法及时记录，实现可以待做。

## Map

Map常用有哪些呢？如图所示：

![Map]({{site.url}}/img/java/Java8-Map-Diagram.png)

## 简介
LinkedHashMap 继承自 HashMap，在 HashMap 基础上，通过维护一条双向链表，解决了 HashMap 不能随时保持遍历顺序和插入顺序一致的问题。

如果不熟悉LinkedHashMap，可以看<a href="https://qin4zhang.github.io/2020/04/04/Java8%E4%B9%8BLinkedHashMap%E4%BB%8B%E7%BB%8D/" target="_blank">Java8之LinkedHashMap介绍</a>

[Java8之LinkedHashMap介绍]({% post_url 2020-04-04-Java8之LinkedHashMap介绍 %})

## 分析

本文主要介绍如何依赖LinkedHashMap实现LRU缓存。

LRU，也即Least Recently Used. 最近最少使用策略的缓存，可以动态删除本地老的数据，保留最新最活跃的热点数据。

## 实现

```
    // LinkedHashMap提供的protected方法，根据实际场景，重写这个方法即可
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
```

## 示例

```

public class LRUCache<K, V> extends LinkedHashMap<K, V> {
  private final int maxEntries;
  private static final int DEFAULT_INITIAL_CAPACITY = 16;
  private static final float DEFAULT_LOAD_FACTOR = 0.75f;

  public LRUCache(int initialCapacity,
                  float loadFactor,
                  int maxEntries) {
    super(initialCapacity, loadFactor, true);
    this.maxEntries = maxEntries;
  }

  public LRUCache(int initialCapacity,
                  int maxEntries) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR, maxEntries);
  }

  public LRUCache(int maxEntries) {
    this(DEFAULT_INITIAL_CAPACITY, maxEntries);
  }

  // not very useful constructor
  public LRUCache(Map<? extends K, ? extends V> m,
                  int maxEntries) {
    this(m.size(), maxEntries);
    putAll(m);
  }

  protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    return size() > maxEntries;
  }

  public static void main(String... args) {
    Map<Integer, String> cache = new LRUCache<>(5);
    for (int i = 0; i < 10; i++) {
      cache.put(i, "hi");
    }
    // entries 0-4 have already been removed
    // entries 5-9 are ordered
    System.out.println("cache = " + cache);

    System.out.println(cache.get(7));
    // entry 7 has moved to the end
    System.out.println("cache = " + cache);

    for (int i = 10; i < 14; i++) {
      cache.put(i, "hi");
    }
    // entries 5,6,8,9 have been removed (eldest entries)
    // entry 7 is at the beginning now
    System.out.println("cache = " + cache);

    cache.put(42, "meaning of life");
    // entry 7 is gone too
    System.out.println("cache = " + cache);
  }

  // 输出为
    cache = {5=hi, 6=hi, 7=hi, 8=hi, 9=hi}
    hi
    cache = {5=hi, 6=hi, 8=hi, 9=hi, 7=hi}
    cache = {7=hi, 10=hi, 11=hi, 12=hi, 13=hi}
    cache = {10=hi, 11=hi, 12=hi, 13=hi, 42=meaning of life}
}

```



## 参考

1. <a href="https://www.javaspecialists.eu/archive/Issue246-LRU-Cache-From-LinkedHashMap.html" target="_blank">LRU Cache From LinkedHashMap</a>

