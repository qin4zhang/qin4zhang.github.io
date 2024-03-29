---
layout:     post
title:      Java8的Stream示例
subtitle:   Java8的函数式Stream示例
date:       2019-08-17
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java
    - Java8 Stream
---
# 注意
> 想法及时记录，实现可以待做。

## 示例介绍
java8提供了stream的流式处理方式，可以非常优雅的处理各种集合操作，远比之前各种循环嵌套要干净清爽许多。

### 一般处理

```
public Map<Long, String> getIdNameMap(List<Account> accounts) {
    return accounts.stream().collect(Collectors.toMap(Account::getId, Account::getUsername));
}
```

### 收集为自身

```
public Map<Long, Account> getIdAccountMap(List<Account> accounts) {
    return accounts.stream().collect(Collectors.toMap(Account::getId, account -> account));
}
```

### 重复key的处理

```
public Map<String, Account> getNameAccountMap(List<Account> accounts) {
    return accounts.stream().collect(Collectors.toMap(Account::getUsername, Function.identity(), (key1, key2) -> key2));
}
```

### 指定具体的收集Map

```
public Map<String, Account> getNameAccountMap(List<Account> accounts) {
    return accounts.stream().collect(Collectors.toMap(Account::getUsername, Function.identity(), (key1, key2) -> key2, LinkedHashMap::new));
}
```

### 分组聚合

```
 Map<Department, List<Employee>> byDept
         = employees.stream()
                    .collect(Collectors.groupingBy(Employee::getDepartment));

```

### 聚合为字符串

```
String joined = things.stream()
                           .map(Object::toString)
                           .collect(Collectors.joining(", "));
```

### 聚合为累加和

```
int total = employees.stream()
                          .collect(Collectors.summingInt(Employee::getSalary)));
```

### 分组聚合后，计算和

```
Map<Department, Integer> totalByDept
         = employees.stream()
                    .collect(Collectors.groupingBy(Employee::getDepartment,
                                                   Collectors.summingInt(Employee::getSalary)));

```

### 复杂类型的分组聚合

```
Map<Tuple, List<BlogPost>> postsPerTypeAndAuthor = posts.stream()
  .collect(groupingBy(post -> new Tuple(post.getType(), post.getAuthor())));
```

### 分组聚合后修改返回值

```
Map<BlogPostType, Set<BlogPost>> postsPerType = posts.stream()
  .collect(groupingBy(BlogPost::getType, toSet()));
```

```
Map<BlogPostType, String> postsPerType = posts.stream()
  .collect(groupingBy(BlogPost::getType, 
  mapping(BlogPost::getTitle, joining(", ", "Post titles: [", "]"))));
```

```
EnumMap<BlogPostType, List<BlogPost>> postsPerType = posts.stream()
  .collect(groupingBy(BlogPost::getType, 
  () -> new EnumMap<>(BlogPostType.class), toList()));
```

### 多个字段分组聚合

```
Map<String, Map<BlogPostType, List>> map = posts.stream()
  .collect(groupingBy(BlogPost::getAuthor, groupingBy(BlogPost::getType)));
```

### 并发分组聚合

```
ConcurrentMap<BlogPostType, List<BlogPost>> postsPerType = posts.parallelStream()
  .collect(groupingByConcurrent(BlogPost::getType));
```


## 参考
1. <a href="https://www.baeldung.com/java-groupingby-collector" target="_blank">Guide to Java 8 groupingBy Collector</a>

