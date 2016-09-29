---
layout: post
title: "JDK 8 中的 ConcurrentHashMap"
date: 2015-08-23 22:21
comments: true
published: false
categories: [Programming]
---

早些时候，我曾经写过一篇 [Java 并发编程之 ConcurrentHashMap](http://blog.khotyn.com/blog/2013/07/12/juc-concurrenthashmap/)，里面分析了 JDK 6 中 ConcurrentHashMap 的实现。最近在看「写给大忙人看的 Java SE 8」，发现 JDK 8 对 Java 并发包中的 ConcurrentHashMap 做了比较多的改进，于是研究了一下改进的内容，写了这篇文章，希望对读者能够有所帮助。

### ConcurrentHashMap 的内部结构

ConcurrentHashMap 的内部结构在 JDK 1.8 中的最大的变化是将原来 Segment 中存放的是一个个的链表。

在 JDK 8 中，ConcurrentHashMap 的内部结构可能会随着数据量的不同而存在不同的形态（实际上 JDK 8 的 HashMap 也是这样的）：

1. 刚开始的时候，一个 ConcurrentHashMap 的 Bin 中的数据是以链表的形式存在的。
2. 当一个链表的数据量大于等于 8（TREEIFY_THRESHOLD）的时候，会将链表转换成一个红黑树。
3. 当一个链表的数据量小于等于 6（UNTREEIFY_THRESHOLD）的时候，会将红黑树重新转换成一个链表。

ConcurrentHashMap 中主要字段的含义：

- `table`：Bin 的数组

#### Node，TreeNode 和 TreeBin

### ConcurrentHashMap 的 put 操作

擦，直接 synchonized(f)？直接对 Hash Bin 做 synchronized？

### ConcurrentHashMap 的 get 操作

### ConcurrentHashMap 的 remove 操作

### ConcurrentHashMap 的 size 操作