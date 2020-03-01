---
title: String有最大长度吗？
date: 2020-03-01 17:54:56
tags:
- Java
categories:
- Java
---
> 这个问题，平时没想过，谁没事干往一个String头上放几个G的data嘛，但是一旦遇到，就问你怎么解决~！

<!-- more -->

# Java String有最大长度吗？
## String内部结构分析

![](/images/20200301/1.png)

看看接口`CharSequence`:

![](/images/20200301/2.png)

打开`java.lang.String`源码，可以发现：
1. 有哪些字段：
    ```
    // 存储字符的容器
    private final char value[];

    // 缓存字符串的hashCode
    private int hash; // Default to 0

    /** use serialVersionUID from JDK 1.0.2 for interoperability */
    private static final long serialVersionUID = -6849794470754667710L;

    /**
     * Class String is special cased within the Serialization Stream Protocol.
     *
     * A String instance is written into an ObjectOutputStream according to
     * <a href="{@docRoot}/../platform/serialization/spec/output.html">
     * Object Serialization Specification, Section 6.2, "Stream Elements"</a>
     */
    private static final ObjectStreamField[] serialPersistentFields =
        new ObjectStreamField[0];
    ```
2. 哪些相关构造器：
    ```
    // 将传入的字符数组，从 value[offset]到 value[offset+count] 这一段复制出来构造一个新的字符数组
    public String(char value[], int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= value.length) {
                this.value = "".value;
                return;
            }
        }
        // Note: offset or count might be near -1>>>1.
        if (offset > value.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        this.value = Arrays.copyOfRange(value, offset, offset+count);
    }
    
    /**
     * 从一个Unicode码数组上面copy一部分创建一个新的字符数组
     * @since  1.5
     */
    public String(int[] codePoints, int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= codePoints.length) {
                this.value = "".value;
                return;
            }
        }
        // Note: offset or count might be near -1>>>1.
        if (offset > codePoints.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
    
        final int end = offset + count;
    
        // Pass 1: Compute precise size of char[]
        int n = count;
        for (int i = offset; i < end; i++) {
            int c = codePoints[i];
            if (Character.isBmpCodePoint(c))
                continue;
            else if (Character.isValidCodePoint(c))
                n++;
            else throw new IllegalArgumentException(Integer.toString(c));
        }
    
        // Pass 2: Allocate and fill in char[]
        final char[] v = new char[n];
    
        for (int i = offset, j = 0; i < end; i++, j++) {
            int c = codePoints[i];
            if (Character.isBmpCodePoint(c))
                v[j] = (char)c;
            else
                Character.toSurrogates(c, v, j++);
        }
    
        this.value = v;
    }
    ```
3. String.copyValueOf()方法：
    ```
    public static String copyValueOf(char data[], int offset, int count) {
        return new String(data, offset, count);
    }
    ```

从以上源码种种迹象
* `length()`返回的是int类型；
* 构造器传入的`count`也是int类型；

，都可以断定，String的长度不会超过`int`的最大值，换言之，`char value[]` 中最多可以保存`Integer.MAX_VALUE`个，即2147483647字符。

但事情是不是这样的呢？

我们需要分编译期和运行期来看：

## 编译期
编译期，也就是从源码角度表面上看，我取一个`Integer.MAX_VALUE`长度的字符串，理论上没问题，因为我没有突破你 int 的最大长度。

但是，实验证明，`String s = "";`中，最多可以有65534个字符。如果超过这个个数。就会在编译期报错。
```
public static void main(String[] args) {
    // 共65534个a，正常编译
    String s = "a...a";
    System.out.println(s.length());

    // 共65535个a，编译报错：错误: 常量字符串过长
    String s1 = "a...a";
    System.out.println(s1.length());
}
```

**明明说好的长度限制是2147483647，为什么65535个字符就无法编译了呢？**

当我们使用字符串字面量直接定义String的时候，是会把字符串在常量池中存储一份的。那么上面提到的65534其实是常量池的限制。
常量池中的每一种数据项也有自己的类型。Java中的UTF-8编码的Unicode字符串在常量池中以CONSTANT_Utf8类型表示。
CONSTANTUtf8info是一个CONSTANTUtf8类型的常量池数据项，它存储的是一个常量字符串。常量池中的所有字面量几乎都是通过CONSTANTUtf8info描述的。CONSTANTUtf8_info的定义如下：
```
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
}
```
由于本文的重点并不是CONSTANTUtf8info的介绍，这里就不详细展开了，我们只需要我们使用**字面量**定义的字符串在class文件中，是使用CONSTANTUtf8info存储的，而CONSTANTUtf8info中有u2 length;表明了该类型存储数据的长度。
u2是无符号的16位整数，因此理论上允许的的最大长度是2^16=65536。而 java class 文件是使用一种变体UTF-8格式来存放字符的，null 值使用两个 字节来表示，因此只剩下 65536－ 2 ＝ 65534个字节。
关于这一点，在the class file format spec中也有明确说明：
> The length of field and method names, field and method descriptors, and other constant string values is limited to 65535 characters by the 16-bit unsigned length item of the CONSTANTUtf8info structure (§4.4.7). Note that the limit is on the number of bytes in the encoding and not on the number of encoded characters. UTF-8 encodes some characters using two or three bytes. Thus, strings incorporating multibyte characters are further constrained.

也就是说，在Java中，所有需要保存在常量池中的数据，长度最大不能超过65535，这当然也包括字符串的定义咯。

## 运行期
上面提到的这种String长度的限制是编译期的限制，也就是使用String s= "";这种字面值方式定义的时候才会有的限制。
那么。String在运行期有没有限制呢，答案是有的，就是我们前文提到的那个Integer.MAX_VALUE ，这个值约等于4G，在运行期，如果String的长度超过这个范围，就可能会抛出异常。(在jdk 1.9之前）
int 是一个 32 位变量类型，取正数部分来算的话，他们最长可以有
```
2^31-1 =2147483647 个 16-bit Unicodecharacter

2147483647 * 16 = 34359738352 位
34359738352 / 8 = 4294967294 (Byte)
4294967294 / 1024 = 4194303.998046875 (KB)
4194303.998046875 / 1024 = 4095.9999980926513671875 (MB)
4095.9999980926513671875 / 1024 = 3.99999999813735485076904296875 (GB)
```
有近 4G 的容量。

## 总结
* 当我们在写代码时使用字面量形式去定义String对象，那么，对象最大长度是：65534（因为它需要被编译进常量池的StringTable当中，JVM编译时对在StringTable中生成的String对象会以CONSTANT_Utf8类型表示，而这个类型有个长度属性是无符号16位整数，而由于一些字符已经占用一些，所以最终剩下：65534个字节）；
* 当我们在运行时接受或动态构造一个String对象，那么，对象最大长度可达：Integer.MAX_VALUE个,即2147483647字符。

# 重温Java基本类型
基本类型--numeric | 占用字节数/byte | 取值范围
--- | --- | ---
byte |1| `[-128, 127]`
short |2| `[-32768,32767]` 或 `$[-2^{15}, 2^{15}-1]$`
int |4|  `[0x80000000, 0x7fffffff]` 或 `$[-2^{31}, 2^{31}-1]$`
long |8| `[0x8000000000000000L, 0x7fffffffffffffffL]` 或 `$[-2^{63}, 2^{63}-1]$`
char |2| `['\u0000', '\uFFFF']`

基本类型--floating-point | 占用字节数/byte | 取值范围
--- | --- | ---
float |4| `[0x0.000002P-126f, 0x1.fffffeP+127f]` 或 `[1.4e-45f, 3.4028235e+38f]`
double |8| `[0x0.0000000000001P-1022, 0x1.fffffffffffffP+1023]` 或 `[4.9e-324,1.7976931348623157e+308]`

> 注意，以上的所有基本类型，都有其对应的封装类型：
> ```
> Number子类：
>     Integer
>     Short
>     Integer
>     Long
>     Float
>     Double
> 
> Character
> ```
> 发现，除了`Character`，其他都是继承`Number`。

# 扩展：Java数组也有最大长度吗？
看了String的源码也知道，其内部是一个`char[]`类型的字符数组，我们调用String的length()函数时，
```
String s = "hhh";
s.length();
```
发现实际其内部是调用了`char[].length`：
```
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
    .......
    public int length() {
        return value.length;
    }
```
而我们又知道字符串在编译期和运行期的不同，那么，可以得知，对于Java数组：
1. 一是规范隐含的限制。Java数组的length必须是非负的int，所以它的理论最大值就是java.lang.Integer.MAX_VALUE = 2^31-1 = 2147483647。
2. 二是具体的实现带来的限制。这会使得实际的JVM不一定能支持上面说的理论上的最大length。
例如说如果有JVM使用uint32_t来记录对象大小的话，那可以允许的最大的数组长度（按元素的个数计算）就会是：
(uint32_t的最大值 - 数组对象的对象头大小) / 数组元素大小
于是对于元素类型不同的数组，实际能创建的数组的最大length也会不同。
JVM实现里可以有许多类似的例子会影响实际能创建的数组大小。