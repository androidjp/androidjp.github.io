---
title: CopyOnWriteArrayList是怎么工作的
date: 2019-11-03 11:42:00
categories:
- Java
tags:
- Java
- Java集合
---
> Java 并发包提供了很多线程安全的集合，有了他们的存在，使得我们在多线程开发下，大大简化了多线程开发的难度，但是如果不知道其中的原理，可能会引发意想不到的问题，所以知道其中的原理还是很有必要的。

<!--more-->

# 背景思想
Copy-On-Write 简称 COW，是一种用于程序设计中的优化策略。其基本思路是，从一开始大家都在共享同一个内容，当某个人想要修改这个内容的时候，才会真正把内容 Copy 出去形成一个新的内容然后再改，这是一种延时懒惰策略。

CopyOnWrite 容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行 Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对 CopyOnWrite 容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以 CopyOnWrite 容器也是一种读写分离的思想，读和写不同的容器。

CopyOnWrite相关两个类：
```
java.util.concurrent.CopyOnWriteArrayList
java.util.concurrent.CopyOnWriteArraySet
```
其中，`CopyOnWriteArraySet`的底层是`CopyOnWriteArrayList`。

本文主要通过源码入手，探讨`CopyOnWriteArrayList`的工作原理。

# 看看源码
## 首先，看看它有什么field：
```
// 可重入锁（不参与对象序列化过程）
final transient ReentrantLock lock = new ReentrantLock();

// 底层对象数组（不参与对象序列化过程，拥有线程可见性）
private transient volatile Object[] array;
```
可以看到：
1. 底层数组使用volatile保证数据可见性和禁止指令重排序；
2. `CopyOnWriteArraySet` 实现线程安全的原理是利用可重入锁。

## 数据修改方法--源码分析
`add()`
```
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock(); //------------- 1. 写操作加锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        //---------------------- 2. 复制数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        //---------------------- 3. 将旧数组 指针指向 新数组
        setArray(newElements);
        return true;
    } finally {
        //---------------------- 4. 解锁
        lock.unlock();
    }
}
```
`set()`
```
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock(); //------------- 1. 写操作加锁
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);
        //---------------------- 2. 如果新元素不等于旧元素，则copy一份副本，更新该位置的元素，最后替换原数组
        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            //------------------ 3. 即使新旧元素相同，但为了触发volatile特性，还是做一次赋值操作
            setArray(elements);
        }
        return oldValue;
    } finally {
        //---------------------- 4. 解锁
        lock.unlock();
    }
}
```
`remove()`
```
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock(); //------------- 1. 写操作加锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            //------------------ 2. 如果是删除数组最后一个元素，那么，直接复制一个长度减1的副本，然后更新原数组
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            //------------------ 3. 否则，分开两次进行复制，再更新原数组
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                                numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        //---------------------- 4. 解锁
        lock.unlock();
    }
}
```

所以，

看到这里，可以得出一些结论：
1. 对于所有的写操作，在多线程对同一个`CopyOnWriteArrayList`对象做写入操作时，是线程安全的。因为用了 `ReentrantLock` 独占锁，保证同时只有一个线程对集合进行修改操作。
2. CopyOnWrite 原来就是指：写时复制。就是在写的时候，先 copy 一个，操作新的对象。然后在覆盖旧的对象，保证 volatile 语义。
2. 占用内存，写时 copy 效率低。我们看这些每一个写操作，都是需要copy数据的。因为 CopyOnWrite 的写时复制机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存，旧的对象和新写入的对象（注意:在复制的时候只是复制容器里的引用，只是在写的时候会创建新对象添加到新容器里，而旧容器的对象还在使用，所以有两份对象内存）。如果这些对象占用的内存比较大，比如说 200M 左右，那么再写入 100M 数据进去，内存就会占用 300M，那么这个时候很有可能造成频繁的 Yong GC 和 Full GC。


## 再看看它的get方法：
`get()`
```
public E get(int index) {
    return get(getArray(), index);
}

final Object[] getArray() {
    return array;
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```
看到 `get()`我们就能知道，“读写分离” 原来就是这么回事：因为的写操作不会直接立即修改原数组，那么，我在读的时候就可以肆无忌惮地并发读，因为我不需要担心数组在读取、遍历过程中会被别人修改。

get 方法很简单。但是会出现一个很致命的问题，那就是一致性问题。

当我们获得了 array 后，由于整个 get 方法没有独占锁，所以另外一个线程还可以继续执行修改的操作，比如执行了 remove 的操作，remove 和 add 一样，也会申请独占锁，并且复制出新的数组，删除元素后，替换掉旧的数组。而这一切 get 方法是不知道的，它不知道 array 数组已经发生了天翻地覆的变化。就像微信一样，虽然对方已经把你给删了，但是你不知道。这就是一个一致性问题。

CopyOnWrite 容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用 CopyOnWrite 容器。


## 它的迭代器：COWIterator
```
static final class COWIterator<E> implements ListIterator<E> {
    /** Snapshot of the array */
    private final Object[] snapshot;
    /** Index of element to be returned by subsequent call to next.  */
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;
    }

    public boolean hasNext() {
        return cursor < snapshot.length;
    }

    public boolean hasPrevious() {
        return cursor > 0;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        if (! hasNext())
            throw new NoSuchElementException();
        return (E) snapshot[cursor++];
    }
}
```
调用 iterator 方法获取迭代器，内部会调用 COWIterator 的构造方法，此构造方法有两个参数，第一个参数就是 array 数组，第二个参数是下标，就是 0。随后构造方法中会把 array 数组赋值给snapshot变量。snapshot 是“快照”的意思，如果 Java 基础尚可的话，应该知道数组是引用类型，传递的是指针，如果有其他地方修改了数组，这里应该马上就可以反应出来，那为什么又会是 snapshot这样的命名呢？没错，如果其他线程没有对 CopyOnWriteArrayList 进行增删改的操作，那么 snapshot 就是本身的 array，但是如果其他线程对 CopyOnWriteArrayList 进行了增删改的操作，旧的数组会被新的数组给替换掉，但是 snapshot 还是原来旧的数组的引用。也就是说 当我们使用迭代器便利 CopyOnWriteArrayList 的时候，不能保证拿到的数据是最新的，这也是一致性问题。

# 总结
**CopyOnWriteArrayList是这样工作的**：
* **写时复制**：在执行`add`、`set`、`remove`等操作修改底层数组时，会先copy一份原数组的副本，然后操作这个副本，操作完毕后，会让原数组的指针指向更新后的副本。
* **写时加锁，写完释放锁**：在执行`add`、`set`、`remove`等操作的开头，就会上一个可重入锁，只有在操作完成后，才会释放锁。并且同一时刻，对于CopyOnWriteArrayList来说，只会执行一个写相关方法，换句话说：线程A 在`add`的时候，线程B 的`set`会等待。
* **读取和遍历的数据，不一定就是最新的数据**：看到其`get()`方法和`COWIterator`迭代器的原理，我们知道，`CopyOnWriteArrayList`只能保证数据的最终一致性，目的是为了牺牲即时一致性，来保证数据的可读。
* **底层数据结构是volatile修饰的数组，所以支持volatile的所有特性**：在之前的文章中笔者有分享volatile的原理，在`CopyOnWriteArrayList`中，volatile关键字的作用则是：保证数据的修改会让刷入主线程，并其他线程的缓存失效，从而达到“可见性”。不过由于“写时复制”特性，只有在最终的原数组指针重新指向新数组的时刻，这个“可见性”才会生效（不过也只有当下一次调用`get()`等读方法时，才会读取最新的数据）。所以，volatile并不能解决数据在多线程中的实时一致性问题，更不能实现原子性。

**CopyOnWriteArrayList 的优点**
* 线程安全
* 读写分离，支持并发读


**CopyOnWriteArrayList 的缺点**
* 由于写操作的时候，需要拷贝数组，会消耗内存，如果原数组的内容比较多的情况下，可能导致 young gc 或者 full gc。
* 不能用于实时读的场景，像拷贝数组、新增元素都需要时间，所以调用一个 set 操作后，读取到数据可能还是旧的，虽然CopyOnWriteArrayList 能做到最终一致性,但是还是没法满足实时性要求。
* 由于实际使用中可能没法保证 CopyOnWriteArrayList 到底要放置多少数据，万一数据稍微有点多，每次 add/set 都要重新复制数组，这个代价实在太高昂了。在高性能的互联网应用中，这种操作分分钟引起故障。
* 内存占用问题：因为CopyOnWrite的写时拷贝机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存，旧的对象和新写入的对象（注意：在拷贝的时候只是拷贝容器里的引用，只是在写的时候会创建新对象添加到新容器里，而旧容器的对象还在使用，所以有两份对象内存）。如果这些对象占用的内存比较大，比如说200M左右，那么再写入100M数据进去，内存就会占用300M，那么这个时候很有可能造成频繁的Yong GC和Full GC。针对内存占用问题，可以通过压缩容器中的元素的方法来减少大对象的内存消耗，比如，如果元素全是10进制的数字，可以考虑把它压缩成36进制或64进制。或者不使用CopyOnWrite容器，而使用其他的并发容器，比如ConcurrentHashMap。
  > 尽量使用批量方法 `addAll()`，添加因为每次添加，容器每次都会进行复制，所以减少添加次数，可以减少容器的复制次数。


**CopyOnWriteArrayList 的设计思想**
* 读写分离，读和写分开（好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。）
* 最终一致性
* 使用另外开辟空间的思路，来解决并发冲突

**CopyOnWriteArrayList 的使用场景**
* **读多写少**的场景。比如白名单，黑名单，商品类目的访问和更新场景，假如我们有一个搜索网站，用户在这个网站的搜索框中，输入关键字搜索内容，但是某些关键字不允许被搜索。这些不能被搜索的关键字会被放在一个黑名单当中，黑名单每天晚上更新一次。当用户搜索时，会检查当前关键字在不在黑名单当中，如果在，则提示不能搜索。



# 参考文章
* https://mp.weixin.qq.com/s?__biz=MzIyODE5NjUwNQ==&mid=2653316286&idx=1&sn=faa5649aefb0958d7bab73b4d4d7c8a0&chksm=f3876d48c4f0e45e20640c8212bcb3a3af35b17cdca774d1171d2a6ad8369ba7f761eabe29f7&scene=7&key=93bab146cfb76d6578a2d711fde6e911546b163716abc2b49706d4e6e754611ca6eb67be498b19fe4ec528918aa01caa6f4a6a116df0f5c004a5374325fb8a6485ec449668d8138f2d84da8de13cf3a1&ascene=0&uin=MjU1MDY2OTI0Mw%3D%3D&devicetype=Windows+10&version=62070152&lang=zh_CN&pass_ticket=pdoJI3%2BSn6V4T3Hfaxyp%2FLAZx6FpYCaKbhu1NtjFtsrhyY9yGKrmq9h33iKr1bxQ
* https://mp.weixin.qq.com/s?__biz=MzU4NjczNzUwOA==&mid=2247483868&idx=1&sn=e33526b478b47e9aa3985c68f2ff5bc6&chksm=fdf7f3d7ca807ac10080e9de204e0cd6138f7a4a47329dd0693a1a89500f0599bad4ce9ce3da&scene=7&key=61615b455c1fa1772017593446b81ea8b0fb9dd848a76e337ec007a64813046ff0b522bac062ae79dea201a32ec18e6204e70f74e619c57772d585ae3f2c00b206ec028abb4ddeb2866db77846e93cc2&ascene=0&uin=MjU1MDY2OTI0Mw%3D%3D&devicetype=Windows+10&version=62070152&lang=zh_CN&pass_ticket=pdoJI3%2BSn6V4T3Hfaxyp%2FLAZx6FpYCaKbhu1NtjFtsrhyY9yGKrmq9h33iKr1bxQ
* https://mp.weixin.qq.com/s?__biz=MzAxOTQxOTc5NQ==&mid=2650496647&idx=2&sn=6466a5e357d2d376110d7f521569afc5&chksm=83c8b97bb4bf306d2f3721b59db673e03467ba870ef1461e8bb150e3cff72a34e16abc25716d&scene=7&key=b872184137e9e0f6870d7355b1c1876c00085e79fbf69638101678b555ea346684701b1eeb37a5d64e847fb84dd90a1b5f33c99de66d1fdc603699e40cbfa645d44c7fb8ccbd347c9928050b7bf49f8e&ascene=0&uin=MjU1MDY2OTI0Mw%3D%3D&devicetype=Windows+10&version=62070152&lang=zh_CN&pass_ticket=pdoJI3%2BSn6V4T3Hfaxyp%2FLAZx6FpYCaKbhu1NtjFtsrhyY9yGKrmq9h33iKr1bxQ