# 1. 类文件结构

我们尝试编译这个`HelloWorld.java`文件：
```
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```
使用到的命令：
```
javac -parameters -d . HelloWorld.java
```
* -parameters： 保留方法参数的名称信息，比如：`main(args)`中的`args`

最终得到 `HelloWorld.class`字节码文件：
* windows用cmder执行以下命令打开：
    ```
    vim HelloWorld.class
    :%!xxd
    ```
* Linus/MacOS也可以执行以下命令打开：
    ```
    od -t xC HelloWorld.class
    ```

```
00000000: cafe babe 0000 0034 001f 0a00 0600 1109  .......4........
00000010: 0012 0013 0800 140a 0015 0016 0700 1707  ................
00000020: 0018 0100 063c 696e 6974 3e01 0003 2829  .....<init>...()
00000030: 5601 0004 436f 6465 0100 0f4c 696e 654e  V...Code...LineN
00000000: cafe babe 0000 0034 001f 0a00 0600 1109  .......4........
00000010: 0012 0013 0800 140a 0015 0016 0700 1707  ................
00000020: 0018 0100 063c 696e 6974 3e01 0003 2829  .....<init>...()
00000030: 5601 0004 436f 6465 0100 0f4c 696e 654e  V...Code...LineN
00000040: 756d 6265 7254 6162 6c65 0100 046d 6169  umberTable...mai
00000050: 6e01 0016 285b 4c6a 6176 612f 6c61 6e67  n...([Ljava/lang
00000060: 2f53 7472 696e 673b 2956 0100 104d 6574  /String;)V...Met
00000070: 686f 6450 6172 616d 6574 6572 7301 0004  hodParameters...
00000080: 6172 6773 0100 0a53 6f75 7263 6546 696c  args...SourceFil
00000090: 6501 000f 4865 6c6c 6f57 6f72 6c64 2e6a  e...HelloWorld.j
000000a0: 6176 610c 0007 0008 0700 190c 001a 001b  ava.............
000000b0: 0100 0b68 656c 6c6f 2077 6f72 6c64 0700  ...hello world..
000000c0: 1c0c 001d 001e 0100 2c63 6f6d 2f65 7861  ........,com/exa
000000d0: 6d70 6c65 2f73 7072 696e 6762 6f6f 7473  mple/springboots
000000e0: 7461 7274 6572 6465 6d6f 2f48 656c 6c6f  tarterdemo/Hello
000000f0: 576f 726c 6401 0010 6a61 7661 2f6c 616e  World...java/lan
00000100: 672f 4f62 6a65 6374 0100 106a 6176 612f  g/Object...java/
00000110: 6c61 6e67 2f53 7973 7465 6d01 0003 6f75  lang/System...ou
00000120: 7401 0015 4c6a 6176 612f 696f 2f50 7269  t...Ljava/io/Pri
00000130: 6e74 5374 7265 616d 3b01 0013 6a61 7661  ntStream;...java
00000140: 2f69 6f2f 5072 696e 7453 7472 6561 6d01  /io/PrintStream.
00000150: 0007 7072 696e 746c 6e01 0015 284c 6a61  ..println...(Lja
00000160: 7661 2f6c 616e 672f 5374 7269 6e67 3b29  va/lang/String;)
00000170: 5600 2100 0500 0600 0000 0000 0200 0100  V.!.............
00000180: 0700 0800 0100 0900 0000 1d00 0100 0100  ................
00000190: 0000 052a b700 01b1 0000 0001 000a 0000  ...*............
000001a0: 0006 0001 0000 0007 0009 000b 000c 0002  ................
000001b0: 0009 0000 0025 0002 0001 0000 0009 b200  .....%..........
```


根据JVM规范，类文件结构如下：
```
ClassFile {
    u4       magic;           // 第0~3个字节：魔数（表示是否是class类型的文件）
    u2       minor_version;   // 2个字节： 小版本号
    u2       major_version;   // 主版本号
    u2       constant_pool_count; // 常量池信息
    cp_info  constant_pool[constant_pool_count-1];
    u2       access_flags;   // 访问修饰（类到底是public还是）
    u2       this_class;     // 本类信息
    u2       super_class;    // 父类信息
    u2       interfaces_count;  // 有多少个接口
    u2       interfaces[interfaces_count]; // 接口信息
    u2       fields_count;
    field_info  fields[fields_count]; // 类变量、成员变量、类静态变量信息
    u2       vmethods_count;
    method_info  methods[methods_count]; // 方法信息
    u2       attributes_count;
    attribute_info  attribute[attributes_count]; //  方法属性信息
}
```

## 1.1 魔数
第0~3个字节：魔数（表示是否是class类型的文件）

好， 我们看到HelloWorld字节码文件第一行：
```
00000000: cafe babe 0000 0034 001f 0a00 0600 1109  .......4........
```
有个这么熟悉的词组，`cafe babe`，意思是什么？ 没错，这就表示的确是 class类型文件

## 1.2 版本
4~7 字节：表示类的版本 

上面的`0000 0034`（52） 表示Java 8


## 1.3 常量池
8~9 字节，表示常量池长度

上面 `001f`（31） 表示常量池有 #1~#30 项， 注意 #0 项 不计入，也没有值。

再往后面看：

`0a00 0600 1109`，其中：
* 第#1项 0a 表示一个 Method 信息；【查下图可以得知】
* `00 06` 和 `00 11(17)`表示它引用了常量池中 #6 和 #17 项来获得这个方法的【所属类】和【方法名】

![](../images/jvm/39.png)


![](../images/jvm/40.png)
