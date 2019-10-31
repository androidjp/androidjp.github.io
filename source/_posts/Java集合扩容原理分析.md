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

> 几个常问问题：
>
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
HashMap | 否 | 元素是单向链表的数组 | 16 | 0.75 | 原容量 x 2
Hashtable | 是 | 元素是单向链表的数组 | 11 | 0.75 | 原容量 x 2 + 1

> 为何是16：16是2^4，可以提高查询效率，另外，32=16<<1
>
> 用移位操作可以极大地提高性能，因为在计算机底层，对位的操作是最方便、最快的。

好，我们接下来用源码来解答几个常见问题。

# 问题一：初始化集合需不需要加初始容量？初始容量设置多大为妙？
答：如果能够大致确定集合中可能存储的元素总量，那么，建议还是根据不同集合的扩容因子来加个初始容量，如：
```
// 我最多存 30个元素
List<String> list = new ArrayList<>(30); // 30 除以 1
// 我最多存 30个键值对
Map<String,String> map = new HashMap<>(40); // 30 除以 0.75
```

# 问题二：HashMap扩容问题
给你一个`new HashMap<>(10000)`，你往里头存放 1w 条缓存数据，此时，你觉得 `HashMap`还会扩容吗？ 是否真的是占用超过75%，就会扩容呢？还是另有答案呢？

> 其实每个集合的扩容思路大同小异，但是实现是不太一样的(比如：`Hashtable`的扩容方法叫做`rehash()`)，这里我们只分析HashMap的扩容原理。

看看HashMap源码，首先看到它的几个默认值：
```
// 默认容量：必须是2的倍数，这里是 16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 如果具有参数的构造函数中的任何一个隐式指定了更高的值，则使用最大容量。
// 取值范围：[2, 1<<30]
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认增量因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```
然后看到它的几个主要的fields：
```
/**
* 链表+数组 的 存储结构 --- table
* 会分配一个2的N次幂 的数值大小，作为table的数组长度
*/
transient Node<K,V>[] table;

/**
* Holds cached entrySet(). Note that AbstractMap fields are used
* for keySet() and values().
*/
transient Set<Map.Entry<K,V>> entrySet;

/**
* 存在于此Map中的所有key-value （键值对） 数量
* 也就是：已占用数
*/
transient int size;

/**
* HashMap结构性改变的次数，包括：
* mapping的数量改变、rehash、用于快速失败（fail-fast） (See ConcurrentModificationException).
*/
transient int modCount;

/**
* 扩容阈值：用于决定是否table需要扩容
* threshold = capacity * load factor
*/
int threshold;

/**
* 加载因子（或者说扩容因子，默认是 0.75）
*/
final float loadFactor;
```
好，我们看看HashMap初始化时做了什么：
```
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
* 将传入的初始容量，计算得到一个 2的n次方 数，作为数组的容量 (table.size)
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
又看到源码里头的不少方法，都会比较当前占用`size`和当前扩容阈值`threshold`，一言不合就要扩容的节奏！

比如这个`putVal()`方法，看看里头的一段code：
```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    // ..........
    ++modCount;
    if (++size > threshold)
        resize();
    // ..........
}
```
后续的update等操作，都会触发`resize()`。我们一睹它的主要逻辑：
```
/**
* resize():    
* 初始化或加倍表大小。 如果为 null，则根据字段阈值中保持的初始容量目标进行分配。
否则，由于我们使用的是 加工后的2的N次幂数，因此每个 bin 中的元素必须保持相同的索引，或者在扩容后的新table中以2的偏移量移动。
* @return the table
*/
final Node<K,V>[] resize() {
    // 1. 拿到现在已存在的table，得到 原来的容量： oldCap
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;

    // 2. 拿到现在已存在的阈值（threshold）： oldThr
    int oldThr = threshold;

    int newCap, newThr = 0;
    // 3. 如果不是第一次创建table，那么，看看是否已经达到最大的可允许范围（1<<30），如果是，那么直接让阈值变成最大的INT值，且不进行扩容。
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
    // 4. 否则，如果 oldCap 乘以 2 ，都不超过 (1<<30)，那么，直接：newCap = oldCap << 1
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold  阈值 乘以 2
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // Default: 容量 = 1<<4， 阈值 = 0.75 * 容量
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    // .........

    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 将 旧table的索引 往 新table 上面搬运。
        // ............
    }
    return newTab;
}
```


HashMap 源码告诉我们：
* table就是那个链表数组的存储结构；
* loadFactor 默认是 0.75；
* initialCapacity 是初始容量，不设置，capacity则为默认16，取值范围是 [2, 1<<30]；
* capacity 一定是 2的N次幂数，通过`tableSizeFor()` 方法保证；
* threshold 是 扩容阈值；
* threshold = table.size * loadFactor；
* HashMap 是否扩容，由 threshold 决定，而 threshold 又由初始容量和 loadFactor 决定；
* `putVal(..)`方法告诉我们：在每次调用`put(k,v)`的过程中，里头会判断当前占用是否大于 threshold，如果大于，则做 扩容；
* `resize()`方法告诉我们：
  * 第一次初始化 容量（capacity）和 阈值（threshold）时：
    * 容量（capacity） = 1 << 4 = 16
    * 阈值（threshold） = 0.75 * 容量（capacity）
  * 后续的扩容：
    * 容量（capacity） = capacity * 2
    * 阈值（threshold） = threshold * 2

那么，回到前面的问题：如果是一个初始入参`capacity=10000`的HashMap，那么，到底会不会在put 了 1w 条记录时 进行扩容？

我们带入源码，会得到：
![](/images/hashMap/hashMap_1.png)

哦，其实此时，我们的table初始容量，并不是10000，而是16xxx这个数。那么，阈值我们口算一下就能得到，13xxx。所以，当我们插入1w条记录，HashMap的这段逻辑：
```
if (++size > threshold)
    resize();
```
这段代码是无法走进来的！！所以，初始给1w的HashMap，往里扔1w条记录，是否会扩容？ **答案是：不会扩容**。

当然，如果给他初始容量设为1000，这又不一样了：
![](/images/hashMap/hashMap_2.png)
好，可以看到，此时，它调整为`1024`，`阈值 = 1024  * 0.75 = 768`，那么，当我们插入1k条记录，那此时就会扩容了~



# 参考文章
* https://juejin.im/post/5db92860e51d4529ee588406