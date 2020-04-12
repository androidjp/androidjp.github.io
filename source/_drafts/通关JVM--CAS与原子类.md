---
title: 通关JVM--CAS与原子类
date: 2020-04-11 21:11:00
categories:
- Java
- JVM
tags:
- JVM
---

<!--more-->

# CAS与原子类
## CAS
CAS 即 Compare and Swap，体现了一种乐观锁思想。

形象例子：
```java
while(true) {
    int 旧值 = 共享变量; // 共享变量此时为 0
    int 结果 = 旧值 + 1; // 结果为 1

    // 此时，共享变量被其他线程改成了 5，那么，本线程的正确结果1 就作废：
    // compareAndSwap 返回 false，重新尝试；
    // compareAndSwap 返回 true，表示本线程做出修改的同时，其他线程并没有干扰此变量；
    if (compareAndSwap(旧值, 结果)) {
        //成功，退出循环
    }
}
```
`compareAndSwap(旧值, 结果)`其实就是 将 旧值 和 当前共享变量的值 再进行一次比较，看是不是一致，不一致就说明有人改动了它。

获取**共享变量**时，为了保证变量的可见性，需要使用 volatile修饰。结合CAS 和 volatile 可以实现无锁并发，适用于竞争不激烈、多核CPU场景。
* 因为没有使用 synchronized，所以线程不会陷入阻塞，这是效率提高的因素之一；
* 但如果竞争激烈，可以想到重试必然频繁发生，反而效率会受影响；

CAS 底层依赖于 一个 Unsafe类来直接调用操作系统底层的CAS指令。

以下是直接使用Unsafe对象 进行线程安全保护的例子：
```java
public class TestCAS {
    public static void main(String[] args) throws InterruptedException {
        DataContainer dataContainer = new DataContainer();
        Thread t1 = new Thread(()-> {
            for (int i = 0; i < 30000; i++) {
                dataContainer.increase();
            }
        });
        Thread t2 = new Thread(()-> {
            for (int i = 0; i < 30000; i++) {
                dataContainer.decrease();
            }
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(dataContainer.getData());
    }
}

class DataContainer {
    private volatile int data;
    static final Unsafe unsafe;
    static final long DATA_OFFSET;

    static {
        // 获取 unsafe 对象（无法直接调用，一定要用反射才能获取）
        try {
            Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
            theUnsafe.setAccessible(true);
            unsafe = (Unsafe) theUnsafe.get(null);
        } catch (NoSuchFieldException | IllegalAccessException e) {
            throw new Error(e);
        }
        // DATA_OFFSET 表示 data 在 DataContainer对象中的偏移量，用于 Unsafe 直接访问该属性
        try {
            DATA_OFFSET = unsafe.objectFieldOffset(DataContainer.class.getDeclaredField("data"));
        } catch (NoSuchFieldException e) {
            throw new Error(e);
        }
    }

    public void increase() {
        int oldValue;
        while (true) {
            // 获取共享变量的旧值
            oldValue = data;
            // CAS 尝试修改 data 为 旧值+1，如果期间旧值被别的线程修改了，返回 false
            if (unsafe.compareAndSwapInt(this, DATA_OFFSET, oldValue, oldValue + 1)) {
                return;
            }
        }
    }

    public void decrease() {
        int oldValue;
        while (true) {
            oldValue = data;
            if (unsafe.compareAndSwapInt(this, DATA_OFFSET, oldValue, oldValue - 1)) {
                return;
            }
        }
    }

    public int getData() {
        return data;
    }
}
```
以上就是自己动手实现的一个基于CAS的原子类。其他和 JDK5引入的juc包中的`AtomicInteger`等类的内部实现原理一致。

## 乐观锁与悲观锁
* CAS 是基于 乐观锁的思想：最乐观的估计，不怕别人来修改共享变量，没修改就好，就算修改了，大不了我再重试呗。
* synchronized 是基于悲观锁的思想：最悲观的估计，得防着其他线程修改共享变量，我上了锁你们都别想改，等到我改完了解开了锁，你们才有机会。不过读倒是可以读。


## 原子操作类
juc（`java.util.concurrent`）中提供了原子操作类，可以提供线程安全的操作，如：`AtomicInteger`,`AtomicBoolean`等，底层都是采用CAS技术+volatile实现。


