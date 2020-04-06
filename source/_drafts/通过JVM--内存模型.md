---
title: 通关JVM--内存模型
date: 2020-04-06 18:40:00
categories:
- Java
- JVM
tags:
- JVM
---

<!--more-->

# 内存模型

## java 内存模型
Java Memory Model（JMM），java 内存模型。

一句话：JMM定义了一套在多线程读写共享数据时（成员变量、数组）时，对数据的可见性、有序性、原子性 的规则和保障。

### JMM如何保证原子性
synchronized(同步关键字)

```java
static Object obj = new Object();

public static void main(....) {
    new Thread(() -> {
        synchronized (obj) {
            ...........
        }
    })
}
```
其中，这个`obj`，在对其加上`synchronized`关键字后，其内存管理会分为几个区域：

![](../images/jvm/82.png)

* Monitor区：每个对象，都有自己的一个monitor区域，翻译为“监视器”，只有用了`synchronized`关键字才有。
* Owner区：Monitor监视器的所有者，同一时刻，只能有一个线程成为Owner；
  * 一旦这个线程调用`monitorenter`成功，即锁定了此对象的monitor区，成为owner，可以跑logic；
  * 跑完logic，调用`monitorexit`成功，本线程释放此锁；
* EntryList：等待区，t2发现t1已经成为了Owner，那么t2就会进入此区域，进入阻塞态，不占用CPU时间；
* WaitSet：Object 的方法 `wait` 和 `notify` 所用到的一个区域；

