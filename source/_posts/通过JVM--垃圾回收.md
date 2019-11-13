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


然后，将这几个.bin文件，于 MAT工具中打开，可以看到：



# 2. 垃圾回收算法

# 3. 分代垃圾回收

# 4. 垃圾回收器

# 5. 垃圾回收调优