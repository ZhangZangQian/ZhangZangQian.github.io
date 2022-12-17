---
title: Java 集合体系学习
author: zhangzangqian
date: 2018-07-10 19:00:00 +0800
categories: [技术]
tags: [Java]
math: true
mermaid: true
---

## 概述

Java 最常见的四种集合为

1. List
2. Set
3. Queue
4. Map

其中，List、Set、Queue 均实现了 Collection 接口，而 Collection 又继承自 Iterable 接口，也就是说，List、Set、Queue 均是可以进行遍历的。而 Map 则是键值对形式的数据结构。

## List

### 特点

1. 有序
2. 可重复
3. 提供了按索引查询的方法


### ArrayList

1. 继承了 AbstractList 类并实现了 List、RandomAccess、Clonable、java.io.Serializable 接口
2. 数组实现
3. 查询快，增、删、改慢
4. 线程不安全

### Vactor

1. 继承了 AbstractList 类并实现了 List、RandomAccess、Cloneable、java.io.Serializable 接口
2. 数组实现
3. 查询快，增、删、改慢
4. 线程安全，每个方法都添加了 synchronized 关键字保证线程安全

### LinkedList

1. 继承了 AbstractSequentialList 类并实现了 List、Deque、Cloneable、java.io.Serializable 接口，而 AbstractSequentialList 类则继承自 AbstractList 类
2. 双向链表实现
3. 查询慢、增、删、改快
4. 线程不安全

## Set

### 特点

1. 无序
2. 不可重复

### HashSet

1. extends AbstractSet implements Set
2. HashMap 实现
3. 无序
4. 线程不安全

### LinkedHashSet

1. extends HashSet implements Set
2. LinkedHashMap 实现
3. 无序
4. 线程不安全

#### TreeSet

1. extends AbstractSet implements NavigableSet
2. TreeMap 实现
3. 通过 Comparator 实现排序
4. 线程不安全

## Queue

### 特点

1. 有序，先进先出
2. 从查询与写入的角度来看，分为阻塞和非阻塞队列
3. 从操作方式来看，分为单向操作跟双向操作（Queue&Deque）
4. 从优先级的角度来看，还分为优先级与非优先级队列

### ConcurrentLinkedQueue

1. 非阻塞队列
2. 线程安全
3. 链表实现
4. 先进先出，无优先级
5. 单向操作

### PriorityQueue

1. 非阻塞队列
2. 线程不安全
3. 数组实现
4. 通过 Comparator 实现排序优先级
5. 单向操作

#### LinkedBlockingQueue

1. 阻塞队列
2. 线程安全
3. 链表实现
4. 先进先出，无优先级
5. 单向操作

### ArrayBlockingQueue

1. 阻塞队列
2. 线程安全
3. 数组实现
4. 先进先出，无优先级
5. 单向操作

### PriorityBlockingQueue

1. 阻塞队列
2. 线程安全
3. 数组实现
4. 通过 Comparator 实现排序优先级
5. 单向操作

### DelayQueue

1. 阻塞队列
2. 线程安全
3. 封装 PriorityQueue 实现
4. 先进先出，无优先级
5. 单向操作

### LinkedTransferQueue

//TODO

### ArrayDeque

1. 非阻塞队列
2. 线程不安全
3. 数组实现
4. 双向操作，从头部或尾部开始操作

### ConcurrentLinkedDeque

1. 非阻塞队列
2. 线程安全
3. 双向链表实现
4. 双向操作，从头部或尾部开始操作

### LinkedBlockingDeque

1. 阻塞队列
2. 线程安全
3. 双向链表实现
4. 双向操作，从头部或尾部开始操作

## Map

### 特点

1. 键值对数据结构

### HashMap

1. 数组链表实现
2. 线程不安全
3. key 允许为空

### LinkedHashMap

### WeakHashMap

### Hashtable

### TreeMap