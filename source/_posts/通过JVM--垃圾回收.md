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

## 标记清除
![](../images/jvm/16.png)

优点：
* 速度快（只需要记录一下哪些空间是空闲的）

缺点：
* 空间不连续，容易产生内存碎片（缺乏空间整理，分配大对象时就很尴尬）

## 标记整理
![](../images/jvm/17.png)

优点：
* 无内存碎片

缺点：
* 牵扯到内存数据的拷贝移动，还需要改变对象引用地址，所以，效率较低。

## 复制
![](../images/jvm/18.png)
![](../images/jvm/19.png)

优点：
* 相对于“标记整理”算法，更快
* 没有内存碎片

缺点：
* 占用双倍的内存空间

# 3. 分代垃圾回收
![](../images/jvm/20.png)
打个比方：
* 整个堆内存：一栋公寓
* 新生代：公寓楼下垃圾存放点
* 老年代：楼上住着的每家每户
* GC：清洁工

那么，清洁工就不需要每次都家家户户走一遍去收集垃圾了，我们每户人家会将真的没用的常见垃圾自己倒到楼下垃圾场中，清洁工每天负责清理楼下的垃圾即可。而家中一些坏椅子等可能还有价值的垃圾，就先放在家中，在家里实在拥挤时，再让清洁工上班清理它。

1. 对象进来，首先暂存在伊甸园
2. 如果发现伊甸园空间不足，那么，触发 Minor GC（复制算法）
   1. 标记 Eden区和 Survivor From区中 要清除的无用对象；
   2. 将仍存活的对象复制到Survivor To 区；
   3. 接着，给Survivor To区的幸存对象，标记寿命加 1（默认，只要幸存区中的对象的寿命达到15次（最大也是15次【对象头存寿命的位置，只有4bit】），就会晋升到老年代中）；
   4. 然后，清理整个Eden区；
   5. 交换 Survivor From 和 Survivor To的内存。（实际上是交换from 和 to 的指针，不会进行数据的拷贝）
3. 如果发现老年代空间不足时，会先触发minor gc，如果仍然空间不足，触发Full GC（对新生代和老年代 都进行一次GC）
4. 如果Full GC后，依然空间不足，那么，触发 OOM Error。

Minor GC 和 Full GC 都会引发 stop the world，暂停所有其他用户线程，等到GC线程清理完垃圾后，其他线程才能恢复运行。（因为对象地址都可能改了，如果这时其他线程还在跑，可能造成大问题）

Full GC 的STW时间更长（因为用的是标记+整理/清除 算法，而且对象本身较多）

## GC相关JVM参数
看另一篇笔记。

## GC分析例子
用以下vm参数去跑一个空的main方法：
```
-Xms20M -Xmx20M -Xmn10M -XX:+UseSerialGC -XX:+PrintGCDetails -verbose:gc
```
> 其中：
> * `+UseSerialGC`: 串行，Young区 和 Old区都使用串行，使用复制算法回收，逻辑简单高效，无线程切换开销

得到的打印结果是：
```
Heap
 def new generation   total 9216K, used 2845K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  34% used [0x00000000fec00000, 0x00000000feec75e0, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 0K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,   0% used [0x00000000ff600000, 0x00000000ff600000, 0x00000000ff600200, 0x0000000100000000)
 Metaspace       used 3374K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 369K, capacity 388K, committed 512K, reserved 1048576K
```
其中：
* `def new generation`：新生代，总大小 9216k（9MB），为什么呢？ 因为 Eden区和Survivor区默认比例是`8:1`，而Survivor to 区被认为是不能用的1MB，所以剩下的总大小就是9MB；
* `tenured generation`：老年代。总大小10MB，还没有被使用到；
* `Metaspace`：元空间

好，那如果我们在main方法中初始化一些对象，看看整个过程：
![](../images/jvm/21.png)

有趣的是，如果我们一开始就给他一个大于等于8MB的对象，那么，它会发现Eden区就算GC也是放不下的，于是他直接选择将对象放到老年代，不触发GC。
![](../images/jvm/22.png)
如果再来一个大对象，那就OOM了：
![](../images/jvm/23.png)

那如果用一个线程去塞两个大对象会怎么样？
![](../images/jvm/24.png)

# 4. 垃圾回收器
1. 串行
   * 单线程（一个保洁工人）
   * 适用场景：堆内存较小（楼层矮）、适合个人电脑
2. 吞吐量优先
   * 多线程
   * 适用场景：堆内存较大、需要多核CPU支持（否则，就是多个线程去争抢时间片，效率就比串行更低）
   * 单位时间内，STW（Stop The World）时间最短。（每次GC虽然时间长一点点，但是每小时只有2次GC）
3. 响应时间优先
   * 多线程
   * 适用场景：同上
   * 尽可能让单次GC的暂定时间变短。STW 时间越短越好。（每次GC虽然时间很短，但是每小时只有N次GC）

## 串行
VM参数：
```
-XX:+UseSerialGC = Serial + SerialOld
```
使用串行，并且，新生代采用复制算法， 老年代采用 标记-整理 算法。

工作模式：
![](../images/jvm/25.png)

## 吞吐量优先
VM参数：
```
// jdk 1.8 下，默认开启
-XX:+UseParallelGC -XX:+UseParallelOldGC
```
同样是：新生代用复制算法，老年代用 标记-整理 算法。 只是变成了“并行”。默认的线程个数等于CPU核数。

当然，当发生GC时，CPU占有率瞬间很高。

工作模式：
![](../images/jvm/26.png)

相关配置：
* `-XX:ParallelGCThreads=n`：设置线程数
* `-XX:+UseAdaptiveSizePolicy`：自动动态调整新生代大小（Eden 和 Survivor的比例，以及整个堆的大小，以及晋升到老年代的阈值，都会动态调整）
* `-XX:GCTimeRatio=ratio`：调整GC时间 与 总时间的 占比。（ratio默认值是 99） 会利用这个公式：`1/(1 + ratio)`来计算比值。 但是因为99是基本无法达到的，不可能100分钟只有1分钟是GC的，所以，一般我们用 19.
* `-XX:MaxGCPauseMillis=ms`：最大暂停毫秒数（默认值是200ms） 注意：堆越大， GC时需要的时间就越多，所以，注意折中。


## 响应时间优先
```
// 简称：CMS 垃圾回收器
-XX:+UseConcMarkSweepGC
```
用于老年代的、并发的、基于 ‘标记-清除’ 算法的 垃圾回收器；

与之相对的，就是适用于新生代的、基于复制算法的垃圾回收器：`-XX:+UseParNewGC`

有时，CMS回收器会发生并发失败问题，此时，就会退化为 `SerialOld`（单线程的、串行的、基于标记-整理的、老年代的垃圾回收器）

特点：可以和用户线程一起执行，不会触发stop-the-world

工作模式：
![](../images/jvm/27.png)
1. 首先，当老年代内存不足时，到达安全点时，会先触发一次“初始标记”；
2. “初始标记” 依然会触发STW，但是持续时间很短，因为此过程只需要标记一些根对象；
3. 接着，用户线程就可以恢复运行，与此同时，GC线程就可以在不触发STW的同时，去标记那些需要被回收的对象；
4. “并发标记” 结束后，来到“重新标记”，此阶段也会触发STW，因为其他线程有可能干预到相关对象，所以要重新标记需要清理的对象；
5. 最后，到了“并发清理”，此时，又可以不用STW了。

并行线程数一般用`-XX:ParallelGCThreads=n`来标记，默认是CPU核数，但是，对于并发线程数的配置，用的是：`-XX:ConcGCThreads=threads`，其中这个`threads`一般设置为CPU核数的 1/4。 因为需要留CPU给到用户线程去干活的。

GC线程对于CPU占用，没用并行这么高，但是，如果考虑到少了一个线程去跑主逻辑，那么， 整个应用的吞吐量，就降低了。

其他相关配置：
* `-XX:CMSInitiatingOccupancyFraction=percent`：执行CMS垃圾回收的内存占比（默认是`60`）；比如设置为：`80`，表示：只要老年代占用达到80%时，就进行CMS垃圾回收，目标就是要预留空间给到正在“并发清理”时其他用户线程可能产生的新对象。
* `-XX:+CMSScavengeBeforeRemark`：是否先进行新生代GC；一个开关，`+`就是打开，`-`就是关闭；
  * 背景：新生代对象可能会引用老年代对象，so，在进行“重新标记”的过程中，实际上，会扫描整个堆，通过扫描新生代每个对象，去做可达性分析，看看有没有用到老年代的这个对象，这个过程对性能影响较大，因为新生代对象多，而且很多新生代对象本身就是垃圾 ，将来本来就要被回收的。
  * 于是：用这个配置，就可以在每次进行“重新标记”之前，都先进行一次 新生代的GC（一般用这个`-XX:+UseParNewGC`来回收）

缺点：将来，内存碎片越来越多，会导致并发失败，此时，就是采用 串行回收，此时，消耗时间就大大提高。

## G1
定义：Garbage First

背景：
* 2004 论文发布
* 2009 JDK6u14 体验
* 2012 JDK 7u4 官方支持
* 2017 JDK 9 默认（取代了CMS回收器）

适用场景：
* 同时注重吞吐量(Throughput) 和 低延迟(Low Latency)，默认暂停目标是200ms，增大暂停目标，会提高吞吐量；
* 超大堆内存，会将堆划分为多个大相等的Region；
* 整体上是 ‘标记-整理’算法，两个区域之间是复制算法；

相关参数：
* `-XX:+UseG1GC`：启用G1GC（JDK8默认不启用）
* `-XX:G1HeapRegionSize=size`：设置上述的`Region`的大小，必须是2的N次幂
* `-XX:MaxGCPauseMillis=time`：最大暂停时间


![](../images/jvm/28.png)
![](../images/jvm/29.png)
![](../images/jvm/30.png)
![](../images/jvm/31.png)

## Full GC
* SerialGC
  * 新生代内存不足发生的垃圾收集- minor gc
  * 老年代内存不足发生的垃圾收集- full ge
* ParallelGC
  * 新生代内存不足发生的垃圾收集- minor gc
  * 老年代内存不足发生的垃圾收集 full gc
* CMS
  * 新生代内存不足发生的垃圾收集· minor gc
  * 老年代内存不足
* G1
  * 新生代内存不足发生的垃圾收集 minor gc
  * 老年代内存不足

CMS 和 G1收集器，它们在老年代内存不足时，需要分情况，不一定就是full GC：
* 当垃圾收集的时间 小于 新垃圾产生的时间，那么，就不算是full GC；
* 当垃圾收集的时间 大于 新垃圾产生的时间，那么，就会退用串行垃圾收集策略，此时占用时间和资源较多，就可以称为full GC。


# 5. 垃圾回收调优