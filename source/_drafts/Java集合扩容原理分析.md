---
title: Java集合扩容原理分析
date: 2019-10-30 21:36:56
tags:
- jdk
- java集合
categories:
- java
---

编程语言提供的集合类，虽然底层还是基于数组、链表这种最基本的数据结构，但是和我们直接使用数组不同，集合在容量不足时，会触发动态扩容来保证有足够的空间存储数据。
动态扩容，涉及到数据的拷贝，是一种「较重」的操作。那如果能够提前确定集合将要存储的数据量范围，就可以通过构造方法，指定集合的初始容量，来保证接下来的操作中，不至于触发动态扩容。


> 到底是 `new ArrayList<>();`简单粗暴好，还是`new ArrayList<>(30);` 给他一个初始容量好呢？万一容量不够会怎么样呢？
>
> 如果给你一个`new HashMap<>(10000)`，你往里头存放 1w 条缓存数据，此时，你觉得 `HashMap`还会扩容吗？ 是否真的是占用超过75%，就会扩容呢？还是另有答案呢？

以上的迷惑，下面给大家快速解答。

# Java集合类齐分析
首先，把常用的Java集合类摆上台，我们先观摩一下他们的主要特征：


集合类 | 线程安全 | 底层 | 数组初始大小 | 扩容因子 | 扩容增量
--- | --- | --- | ---| ---| ---
ArrayList | 否 | 数组 | 10 byte | 1【当元素个数超过容量长度的100%时，进行扩容】 | 原容量 x 2
Vector    | 是 | 数组 |  10 byte | 1 | 原容量 x 1.5 + 1
HashSet  | 否 | HashMap | 16 | 0.75 | 原容量 x 2
HashMap | 否 | | 16 | 0.75 | 原容量 x 2
Hashtable | 是

> 为何是16：16是2^4，可以提高查询效率，另外，32=16<<1
>
> 用移位操作可以极大地提高性能，因为在计算机底层，对位的操作是最方便、最快的。

好，我们接下来用源码来解答几个常见问题。

# 问题一：初始化集合需不需要加初始容量？
```

```

# 问题二：初始容量设置多大为妙？

# 问题三：HashMap扩容问题
```
// 默认容量：必须是2的倍数，这里是 16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 如果具有参数的构造函数中的任何一个隐式指定了更高的值，则使用最大容量。
// 取值范围：[2, 1<<30]
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认增量因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//--------
// 构造器
//--------
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
    this.threshold = tableSizeFor(initialCapacity);
}

// ...........

/**
* 计算得到一个 2的n次方 数，作为数组的容量
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

// ...........

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    // ..........
    ++modCount;
    if (++size > threshold)
        resize();
    // ..........
}

```
HashMap 源码告诉我们：
* loadFactor 默认是 0.75
* initialCapacity 是初始容量，不设置，则为默认16，取值范围是 [2, 1<<30]
* threshold 是 扩容阈值
* threshold = table.size * loadFactor
* HashMap 是否扩容，由 threshold 决定，而 threshold 又由初始容量和 loadFactor 决定。

# 参考文章
* https://juejin.im/post/5db92860e51d4529ee588406