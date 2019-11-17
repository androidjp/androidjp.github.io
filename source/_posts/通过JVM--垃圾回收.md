---
title: 通关JVM--垃圾回收
date: 2019-11-13 18:26:00
categories:
- Java
- JVM
tags:
- JVM
---

<!--more-->

# 1. 如果判断对象可以回收

## 1.1. 引用计数法
早期的Python虚拟机的GC，用的就是这个算法。

如果对象被其他对象所引用，那么，它的引用数大于0，此时，对象无法进行GC。

缺点：遇到循环引用，凉凉！！

![](../images/jvm/11.png)

## 1.2. 可达性分析算法
JVM的垃圾回收器，采用的就是这个“可达性分析”来探索所有存活的对象。

如果一个对象，没有给根对象引用，那么，就可以进行GC。

扫描堆中的对象，看是否能在 以 GC Root对象 为起点的引用链 找到该对象，找不到，表示可以回收。


### 哪些对象可以作为GC Root？

这里有个内存分析工具：MAT [官网](http://www.eclipse.org/mat/)

下面看个代码例子：
```
public static void main(String[] args) throws IOException {
    List<String> list = new ArrayList<>();
    list.add("A");
    list.add("B");
    System.out.println(1); // status: 1
    System.in.read();
    list = null;
    System.out.println(2); // status: 2
    System.in.read();
    System.out.println("end");
}
```
然后，通过以下命令，生成代码调用栈信息到.bin文件中：
```sh
// 查看已启动的Java进程
jps

// 导出运行时栈信息，将其转成格式是"b"即二进制、且"live"即只显示还没被回收掉的对象（存活对象），文件名为 "1.bin"
jmap -dump:format=b,live,file=1.bin <Java进程ID>
```


然后，将这几个.bin文件，于 MAT工具中打开，就可以查看此时刻所有的CG Root对象：
![](../images/jvm/12.png)
![](../images/jvm/13.png)
好，那么，线程运行时，每次方法调用，都会生成一个栈帧，也就是说，每个栈帧内生成的局部变量所引用的对象，都可以成为一个CG Root对象。（注意：不是指 这个局部变量，因为它只是一个引用，而指的是这个变量所引用的堆中的对象）

## 五种引用类型
![](../images/jvm/14.png)
* 强引用：平常用的对象间引用，都是强引用；沿着GC Root链，能够找到它，它那就是强引用，如：A1。只有当所有链都断开了，A1对象才能被回收。
* 软引用（SoftReference）：当B对象和A2对象之间断开，那么，满足以下全部条件，A2对象 是能够被回收的：
  * 没有Root对象对其强引用
  * 正在进行GC
  * 回收完发现还是内存不足时
* 弱引用（WeakReference）： 和软引用的回收条件，唯一区别：弱引用是无论内存是否足够，一定会被回收。
  > ![](../images/jvm/15.png)
  > 当软引用或者弱引用的对象被GC后，因为它们本身也是对象，也需要占内存，所以，会随后进入 引用队列中，等待进一步的操作，如：引用队列遍历进行GC。
* 虚引用（PhantomReference）：
  * 必须配合引用队列使用。
  * 内部生成Cleaner对象，并记录直接内存地址，为了是在其引用的`ByteBuffer`对象被GC后，能够通过入队，并遍历看是否有新的Cleaner，来达到进一步回收“直接内存”的目的。
* 终结器引用（FinalReference）：
  * 必须配合引用队列使用
  * 对象回收过程：
    1. 当A4对象继承Object并重写了`finallize()`方法时，并且又没有Root对象强引用它时；
    2. 此时，A4对象不会立即被回收，JVM会将终结器引用对象入队；
    3. 然后，入队后的终结器引用依然连着A4对象；
    4. 随后，再由一个名为“finallize”的、优先级很低的线程，去查看这个引用队列中，是否存在终结器引用，如果发现，就会通过这个终结器引用，找到A4对象，调用其`finallize()`方法；
    5. 完了之后，在下一次GC触发时，A4对象才会被最终释放内存。
  * 不推荐用它来释放资源。

## 软链接回收例子
看看下面代码：
```
public static final int _4MB = 1024*1024<<2;
private static void soft() throws IOException {
    List<SoftReference<byte[]>> list = new ArrayList<>();
    for (int i = 0; i < 5; i++) {
        SoftReference<byte[]> ref = new SoftReference<>(new byte[_4MB]);
        list.add(ref);
        System.out.println(String.format("第%d次加入list后 ---------------", i+1));
        for (SoftReference<byte[]> item : list) {
            System.out.println(item.get());
        }
    }
    System.out.println("end.................");
    for (SoftReference<byte[]> softReference : list) {
        System.out.println(softReference.get());
    }
}
```
我们在启动时给他以下启动参数：最大堆内存20MB，并打印垃圾回收详细信息
```
-Xmx20m -XX:+PrintGCDetails -verbose:gc
```
最终运行结果：
```
第1次加入list后 ---------------
[B@f6f4d33
第2次加入list后 ---------------
[B@f6f4d33
[B@23fc625e
第3次加入list后 ---------------
[B@f6f4d33
[B@23fc625e
[B@3f99bd52
[GC (Allocation Failure) [PSYoungGen: 3305K->480K(6144K)] 15593K->13397K(19968K), 0.0011852 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Ergonomics) [PSYoungGen: 480K->0K(6144K)] [ParOldGen: 12917K->13273K(13824K)] 13397K->13273K(19968K), [Metaspace: 3960K->3960K(1056768K)], 0.0068424 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
第4次加入list后 ---------------
[B@f6f4d33
[B@23fc625e
[B@3f99bd52
[B@4f023edb
[Full GC (Ergonomics) [PSYoungGen: 4208K->4096K(6144K)] [ParOldGen: 13273K->13214K(13824K)] 17482K->17310K(19968K), [Metaspace: 3960K->3960K(1056768K)], 0.0061849 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (Allocation Failure) [PSYoungGen: 4096K->0K(6144K)] [ParOldGen: 13214K->889K(7680K)] 17310K->889K(13824K), [Metaspace: 3960K->3960K(1056768K)], 0.0055080 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
第5次加入list后 ---------------
null
null
null
null
[B@3a71f4dd
end.................
null
null
null
null
[B@3a71f4dd
Heap
 PSYoungGen      total 6144K, used 4376K [0x00000000ff980000, 0x0000000100000000, 0x0000000100000000)
  eden space 5632K, 77% used [0x00000000ff980000,0x00000000ffdc63d0,0x00000000fff00000)
  from space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
  to   space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
 ParOldGen       total 7680K, used 889K [0x00000000fec00000, 0x00000000ff380000, 0x00000000ff980000)
  object space 7680K, 11% used [0x00000000fec00000,0x00000000fecde5e0,0x00000000ff380000)
 Metaspace       used 3973K, capacity 4646K, committed 4864K, reserved 1056768K
  class space    used 435K, capacity 462K, committed 512K, reserved 1048576K
```
解释一下上面的输出log：
1. 首先，前三次循环，没有问题；
2. 到了第四次循环，内存已经很紧张了，尝试进行一次Minor GC `GC (Allocation Failure)`:
   ```
   [GC (Allocation Failure) [PSYoungGen: 3402K->480K(6144K)] 15690K->13423K(19968K), 0.0014941 secs]
   新生代从 3MB多 回收到了 480KB
   ```
   然后又触发了一次Full GC
   ```
   [Full GC (Allocation Failure) [PSYoungGen: 4096K->0K(6144K)] [ParOldGen: 13214K->889K(7680K)] 17310K->889K(13824K), [Metaspace: 3960K->3960K(1056768K)], 0.0055080 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
   ```
   循环结束后，4个 软引用关联的对象依然没有被回收；
3. 到了第五次循环，直接先触发full GC：
   ```
   [Full GC (Ergonomics) [PSYoungGen: 4208K->4096K(6144K)] [ParOldGen: 13273K->13214K(13824K)] 17482K->17310K(19968K), [Metaspace: 3960K->3960K(1056768K)], 0.0061849 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
   ```
   full GC之后，发现还是没有释放掉什么空间，于是，触发了下面的软引用的GC：
   ``` 
   [Full GC (Allocation Failure) [PSYoungGen: 4096K->0K(6144K)] [ParOldGen: 13214K->889K(7680K)] 17310K->889K(13824K), [Metaspace: 3960K->3960K(1056768K)], 0.0055080 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
   ```
   这样一来，此时，在塞第5个`byte[]`时，前4个`byte[]`由于软引用回收时机被触发（条件：GC之后还是内存不足），所以，在塞第5个之前，前4个都被回收掉了。


好，那代码走到最后，对象是释放掉了，但是，软引用本身自己要怎么释放呢？ 利用引用队列

改一下上面例子的code：
```
 private static void soft() throws IOException {
    List<SoftReference<byte[]>> list = new ArrayList<>();
    // 1. 引用队列
    ReferenceQueue<byte[]> queue = new ReferenceQueue<>();

    for (int i = 0; i < 5; i++) {
        // 2. 关联引用队列，当关联的byte[]被回收，软引用自己会加入到queue中
        SoftReference<byte[]> ref = new SoftReference<>(new byte[_4MB], queue);
        list.add(ref);
        System.out.println(String.format("第%d次加入list后 ---------------", i+1));
        for (SoftReference<byte[]> item : list) {
            System.out.println(item.get());
        }
    }
    System.out.println("end.................");
    // 3. 遍历queue，找到，则删除，直到删除完毕
    Reference<? extends byte[]> poll = queue.poll();
    while (poll != null) {
        list.remove(poll);
        poll = queue.poll();
    }
}
```
这样，就能在关联对象被回收时，软引用对象自身也会被加入到这个队列中。

## 弱引用回收实例
```
public static final int _4MB = 1024*1024<<2;

public static void main(String[] args) throws IOException {
    List<WeakReference<byte[]>> list = new ArrayList<>();
    // 1. 引用队列
    ReferenceQueue<byte[]> queue = new ReferenceQueue<>();

    for (int i = 0; i < 5; i++) {
        // 2. 关联引用队列，当关联的byte[]被回收，软引用自己会加入到queue中
        WeakReference<byte[]> ref = new WeakReference<>(new byte[_4MB], queue);
        list.add(ref);
        System.out.println(String.format("第%d次加入list后 ---------------", i+1));
        for (WeakReference<byte[]> item : list) {
            System.out.println(item.get());
        }
    }
    System.out.println("end.................");
    // 3. 遍历queue，找到，则删除，直到删除完毕
    Reference<? extends byte[]> poll = queue.poll();
    while (poll != null) {
        list.remove(poll);
        poll = queue.poll();
    }
}
```
好，同样打印结果：
```
第1次加入list后 ---------------
[B@f6f4d33
第2次加入list后 ---------------
[B@f6f4d33
[B@23fc625e
第3次加入list后 ---------------
[B@f6f4d33
[B@23fc625e
[B@3f99bd52
[GC (Allocation Failure) [PSYoungGen: 3305K->488K(6144K)] 15593K->13362K(19968K), 0.0014076 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Ergonomics) [PSYoungGen: 488K->0K(6144K)] [ParOldGen: 12874K->986K(12800K)] 13362K->986K(18944K), [Metaspace: 3950K->3950K(1056768K)], 0.0050504 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
第4次加入list后 ---------------
null
null
null
[B@4f023edb
第5次加入list后 ---------------
null
null
null
[B@4f023edb
[B@3a71f4dd
end.................
Heap
 PSYoungGen      total 6144K, used 4265K [0x00000000ff980000, 0x0000000100000000, 0x0000000100000000)
  eden space 5632K, 75% used [0x00000000ff980000,0x00000000ffdaa568,0x00000000fff00000)
  from space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
  to   space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
 ParOldGen       total 12800K, used 5082K [0x00000000fec00000, 0x00000000ff880000, 0x00000000ff980000)
  object space 12800K, 39% used [0x00000000fec00000,0x00000000ff0f6840,0x00000000ff880000)
 Metaspace       used 3957K, capacity 4646K, committed 4864K, reserved 1056768K
  class space    used 435K, capacity 462K, committed 512K, reserved 1048576K
```
与软引用唯一不同的是，full GC在发生之后，所有的前面被弱引用引用的对象，都被回收掉了。


# 2. 垃圾回收算法

# 3. 分代垃圾回收

# 4. 垃圾回收器

# 5. 垃圾回收调优