---
title: 源码学习 - HashMap
tags:
  - 源码解析
categories: Java
updated: 1638199710000
date: 2021-11-29 23:28:47
---

> Map是什么,可以是一个键到值的Object,可以是一个键值对的集合, 是函数抽象的数学模型

似不似?

HashMap就是Map实现的佼佼者,它使用哈希表作为底层数据结构

> 注意哦,作为性能的最上层,HashMap是不考虑线程安全的,这是上层人的优越,多线程情况下,可以选用`ConcurrentHashMap` > `Collections.synchronizedMap(HashMap<>())`

<!-- more -->
### 结构

> 当Node数组的长度大于8时,会转变红黑树来存储

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
  // 存储数据的Node数组,一枪秒了有木有,一看就是链表结构 
  transient Node<K,V>[] table;   
}
```

![Iz66S0.png](https://z3.ax1x.com/2021/11/22/Iz66S0.png)


### 解析

#### HashMap()

> 初始化部分主要决定当前容量(loadFactor ),以及存储的阀值(loadFactor)

```java
// 最大容量，当两个构造函数中任何一个带参数的函数隐式指定较大的值时使用。必须是2的幂<= 1<<30
static final int MAXIMUM_CAPACITY = 1 << 30;
...
// 默认扩容比例
static final float DEFAULT_LOAD_FACTOR = 0.75f;
public HashMap(int initialCapacity) {  
    this(initialCapacity, DEFAULT_LOAD_FACTOR);  
}
...
public HashMap(int initialCapacity, float loadFactor) {
    // 针对煞笔的处理,不会真的有人给0吧
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity:" +initialCapacity);
    // 当前最大容量    
    if (initialCapacity > MAXIMUM_CAPACITY) initialCapacity = MAXIMUM_CAPACITY;
    // 判断扩容比例
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +loadFactor);
    this.loadFactor = loadFactor;
    // 决定下一次调整容量的大小(临界值)
    this.threshold = tableSizeFor(initialCapacity);
}
// 返回给定目标容量的2次幂大小
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

使用指定的初始容量和默认的加载因子来初始化HashMap。这里应该注意的是，有时它不是您指定的初始容量。例如新HashMap (20,0.8);那么实际的初始容量是32，因为tablesize()方法严格要求初始容量增加到2的幂，只能是16、32、64、128