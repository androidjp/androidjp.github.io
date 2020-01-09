---
title: Java中的boolean占多少字节
date: 2020-01-09 21:31:23
tags:
- java
categories:
- java
---

> 引言：网上众说纷纭：1个字节；1位；4 byte....... 到底信哪个？ 这篇文章快速为你解答。

<!--more-->

# 先回答你
boolean 占 **4 byte**，相当于 int 的空间占用；

boolean[] 内每个元素 占 **1 byte**，相当于 byte 的空间占用；

Boolean 属于对象类型，其内部唯一的成员变量就是一个boolean值（其余都是静态变量），理论上类实例空间占用也是 **4 byte**

# 看看官方怎么说

官方文档：
> boolean: The boolean data type has only two possible values: true and false. Use this data type for simple flags that track true/false conditions. This data type represents one bit of information, but its "size" isn't something that's precisely defined.

上面的说的很清楚，boolean 值只有 true 和 false 两种，这个数据类型只代表 1 bit 的信息，但是它的“大小”没有严格的定义。也就是说，不管它占多大的空间，只有一个bit的信息是有意义的。

JVM规范上：
> Although the Java Virtual Machine defines a boolean type, it only provides very limited support for it. There are no Java Virtual Machine instructions solely dedicated to operations on boolean values. Instead, expressions in the Java programming language that operate on boolean values are compiled to use values of the Java Virtual Machine int data type.

Java 虚拟机虽然定义了 boolean 类型，但是支持是有限的，没有专门的虚拟机指令。在 Java 语言中，对 boolean 值的操作被替换成 int 数据类型。

> The Java Virtual Machine does directly support boolean arrays. Its newarray instruction (§newarray) enables creation of boolean arrays. Arrays of type boolean are accessed and modified using the byte array instructions baload and bastore (§baload, §bastore).

Java 虚拟机没有直接支持 boolean 数组。boolean 类型数组和 byte 数组公用指令。

> In Oracle’s Java Virtual Machine implementation, boolean arrays in the Java programming language are encoded as Java Virtual Machine byte arrays, using 8 bits per boolean element.

在 Oracle 的 Java 虚拟机实现中，Java 语言中的 boolean 数组被编码成 Java 虚拟机的 byte 数组，每个元素占 8 比特。

> The Java Virtual Machine encodes boolean array components using 1 to represent true and 0 to represent false . Where Java programming language boolean values are mapped by compilers to values of Java Virtual Machine type int , the compilers must use the same encoding.

Java 虚拟机使用 1 表示 true，0 表示 false 来编码 boolean 数组。Java 语言的 boolean 值被编译器映射成 Java 虚拟机的 int 类型的时候，也是采用一样的编码。

**简单来说**：

说占一个字节的人是这样认为的：计算机处理数据的最小单位是字节，一个字节等于8位，存储true和false的实际存储空间是：用1一个字节的最低位存储，其他位用0填补。虽然编译后的Boolean类型是0和1，但是依旧需要8位来进行存储，true的存储为：00000001；false的存储为：00000000。

认为占4个字节的理由是：在Java虚拟机中没有任何字节码指令供boolean值专用，Java中的boolean值，在编译后都使用Java虚拟机中的int类型来代替，而boolean数组会被编码成Java虚拟机中的byte数组，数组中的每个元素占8位（1字节），由此，可以得出boolean类型在单独使用时占4个字节，作为boolean数组存在的时候是一个字节。

之所以在虚拟机中将boolean类型使用int类型来代替的原因是CPU硬件来说的，转换成int对于32位的CPU来说，具有更高的存取效果。


# 从源码角度分析
我们跑一下下面代码：
```
public class BoolAndBoolArray {
    public static void main(String[] args) {
        boolean x = true;
        boolean y;
        boolean[] z = new boolean[4];
        System.out.println(x);
        for (int i = 0; i < 3; i++) {
            System.out.print(z[i]);
        }
    }
}
```
编译然后反编译
```
javac BoolAndBoolArray.java
javap -v BoolAndBoolArray.class
```

看到指令：
```
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=5, args_size=1
         0: iconst_1                         // 加载常量池中int值 1 --- 表示 true
         1: istore_1                         // 将常量池中int值 1 存入 slot[1] 中 （这里的slot 指的是 存储本地变量的数组）
         2: iconst_4                         // 加载常量池中int值 4
         3: newarray       boolean           // 创建boolean[4]数组
         5: astore_3                         // 将boolean[4]数组放入 slot[3]中
         6: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         9: iload_1                           // 打印 x 值，也就是 slot[1]的值
        10: invokevirtual #3                  // Method java/io/PrintStream.println:(Z)V
        13: iconst_0                          // 加载常量池中的int值 0
        14: istore        4                   // 将其放入 slot[4]
        16: iload         4                   // 将 slot[4] 的值 又加载到 操作数栈
        18: iconst_3                          // 加载常量池中int值 3 到 操作数栈
        19: if_icmpge     38                  // 结束循环的标记：i值>=3
        22: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        25: aload_3                           // 将 boolean[4]数组 加载进操作数栈
        26: iload         4                   // 将 i 载入操作数栈
        28: baload                            // load出byte元素
        29: invokevirtual #4                  // Method java/io/PrintStream.print:(Z)V
        32: iinc          4, 1                // i++
        35: goto          16
        38: return
```

由字节码指令就可以证明以上所述：
1. JVM指令中没有一条说是 ‘加载boolean值’ 的指令，只有 `iload` 和 `baload` 分别表示 加载 int类型  以及 byte类型 的值；
2. JVM用 int 的 1 表示 true，0 表示 false；
3. JVM让 boolean型的变量操作，和 int型共用 一套指令集，在JVM眼里，boolean型也就是int型；
4. JVM让 boolean数组 中的元素操作 使用 byte相关指令，说明boolean数组中元素大小等同于 byte大小 --- 1 byte；


# 参考文章
* [Java中的boolean占多少个字节？](https://mp.weixin.qq.com/s?subscene=23&__biz=Mzg2OTA4Mjg5NQ==&mid=2247483669&idx=1&sn=9555df728fc4ec6a7160a64469fe7466&chksm=cea33375f9d4ba632a08f57fc221af9a69f6112c9bf22db954acd443ec541c834453c196e8cc&scene=7&key=69038692c09c4f48bc5af16571e5da8d0c49523b88c24758ea6772e21dec12693fb8c2ab383ff369284827d4cc1b32ffd2241ad1d77837b11eaaf1d477636694c8dec517b1d0dd147a1cb84a47ec2a30&ascene=0&uin=MjU1MDY2OTI0Mw%3D%3D&devicetype=Windows+10&version=62070158&lang=zh_CN&exportkey=A1DZSaSJy5alXMjHOOx%2B33o%3D&pass_ticket=H3eYA3Bz9co9X41uSLGzoACLUbwKFfkfHF3b0zU8m%2FxJ6K0hIwZN5UYqKKUYfymj)
* [在Java中boolean类型占多少字节？大多数人都回答错了...](https://mp.weixin.qq.com/s?subscene=23&__biz=MzIzMzgxOTQ5NA==&mid=2247490231&idx=2&sn=67fcbdd631d98cf5982a2d57ce2c2afd&chksm=e8fe86bedf890fa85144c478a35bbd302587ab13d843c29d953c170742973931e104241e23c2&scene=7&key=aa7656a594d1173368ca896d7826918aeddd65bd1fe537143622380fb6c6af942a50f2c28cf7254e8f6bb270adac956445bce36fda162d7fe00c6ae470bc73f1d5c0a499e817f1148da12ba07bbcdf87&ascene=0&uin=MjU1MDY2OTI0Mw%3D%3D&devicetype=Windows+10&version=62070158&lang=zh_CN&exportkey=A4wSZoM4XsyS8RgPH%2BkTnyE%3D&pass_ticket=H3eYA3Bz9co9X41uSLGzoACLUbwKFfkfHF3b0zU8m%2FxJ6K0hIwZN5UYqKKUYfymj)
* https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5