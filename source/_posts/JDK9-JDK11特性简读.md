---
title: JDK9~JDK11特性简读
date: 2019-10-27 16:07:48
tags:
- jdk
categories:
- java
---
> JDK11 发布时间：2018/9/25。Java 11和2017年9月份发布的Java 9以及 2018年3月份发布的Java 10相比，其最大的区别就是：在长期支持(Long-Term-Support)方面，Oracle表示会对Java 11提供大力支持，这一支持将会持续至2026年9月。这是据 Java 8 以后支持的首个长期版本。

# 新特性 （JDK 9-11）
### 局部变量类型推断
```
var javaStack = "hello";
System.out.println(javaStack);
```
局部变量类型推断就是左边的类型直接使用 var 定义，而不用写具体的类型，编译器能根据右边的表达式自动推断类型，如上面的 String 。
```
var javaStack = "hello";
相当于
String javaStack = "hello";
```
### 字符串增强
```
// 判断字符串是否为空白

" ".isBlank();                  //true

// 去除首尾空格

" Javastack ".strip();          //"Javastack"

// 去除尾部空格

" Javastack ".stripTrailing();  // " Javastack"

// 去除首部空格

" Javastack ".stripLeading();   // "Javastack "

// 复制字符串

"Java".repeat(3);               //"JavaJavaJava"

// 行数统计
"A\nB\nC".lines().count();      // 3
```
### 集合加强(不可变集合)
> Java 9 开始，Jdk里面为集合（List/Set/Map）都添加了of和copyOf方法，这两个方法都用来创建**不可变集合**。

例子1：
```
var list = List.of("java", "Python", "C");
var copy = List.copyOf(list);
System.out.println(list == copy);   // true
```
例子2：
```
var list = new ArrayList<String>();
var copy = List.copyOf(list);
System.out.println(list == copy);   // false
```
> copyOf方法会先判断来源集合是否是`AbstractImmutableList`类型的，如果是，直接返回，否则，会自动调用`of`创建一个新集合。

例子2中，`list`不属于不可变 `AbstractImmutableList` 类的子类，所以`copyOf`方法又创建了一个新的实例。

> 注意：使用 `of` 和 `copyOf` 创建的集合为不可变集合，不能进行添加、删除、替换、排序等操作，不然会报 `java.lang.UnsupportedOperationException` 异常。

### Stream加强
Stream 是JDK 8 的新特性，而 JDK 9 开始对Stream新增加了以下4个方法：
1. 增加单个参数构造方法，可为null
   ```
   Stream.ofNullable(null).count();
   ```
2. takeWhile 和 DropWhile  两个截止执行方法
   ```
   Stream.of(1,2,3,2,1)
     .takeWhile(n -> n < 3)
     .collect(Collectors.toList()); // [1,2]
   ```
   以上例子，从读到'3'时不满足条件，就已经截止执行了。
   ```
   Stream.of(1,2,3,2,1)
     .dropWhile(n -> n < 3)
     .collect(Collectors.toList()); // [3,2,1]
   ```
   第二个例子，就是从 `n < 3` 不成立的那一刻开始算。
3. iterate重载
   
   这个 iterate 方法的新重载方法，可以让你提供一个 Predicate (判断条件)来指定什么时候结束迭代。

### Optional 加强
Opthonal 也增加了几个非常酷的方法，现在可以很方便的将一个 Optional 转换成一个 Stream, 或者当一个空 Optional 时给它一个替代的。
```
Optional.of("javaStack").orElseThrow();    // javaStack
Optional.of("javaStack").stream().count(); // 1
Optional.ofNullable(null)
   .or(() -> Optional.of("javaStack"))
   .get();                                // javaStack
```
### InputStream 加强
```
// transferTo，可以用来将数据直接传输到 OutputStream，这是在处理原始数据流时非常常见的一种用法
var classLoader = ClassLoader.getSystemClassLoader();
var inputStream = classLoader.getResourceAsStream("javaStack.txt");
var newFile = File.createTempFile("javaStack2", "txt");
try (var outputStream = new FileOutputStream(newFile)) {
   // inputStream 读取数据，然后丢到新文件中
   inputStream.transferTo(outputStream);
}
```
### HTTP Client API (代替Apache的HttpClient的节奏)
这是 Java 9 开始引入的一个处理 HTTP 请求的的孵化 HTTP Client API，该 API 支持同步和异步，而在 Java 11 中已经为正式可用状态，你可以在 `java.net`包中找到这个 API。
```
var request = HttpRequest.newBuilder()
    .uri(URI.create("https://javastack.cn"))
    .GET()
    .build();
    
var client = HttpClient.newHttpClient();

// 同步
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.body());

// 异步
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println);
```

### 化繁为简，一个命令编译运行源码
```
// 编译
javac Demo.java

// 运行
java Demo
```
变成
```
// JDK 11
java Demo.java
```
# 更多新特性
> 这是官方发布的 Java 11 包含的所有新特性，提供17个JEP（JDK Enhancement Proposal 特性增强提议）：
> 
> This release includes seventeen features:
> * 181: Nest- Based Access Control
> * 309: Dynamic Class-File Constants
> * 315: Improve Aarch64 Intrinsics
> * 318: Epsilon: A No-Op Garbage Collector (Experimental)
> * 320: Remove the Java EE and CORBA Modules
> * 321: HTTP Client (Standard)
> * 323: Local-Variable Syntax for Lambda Parameters
> * 324: Key Agreement with Curve25519 and Curve448
> * 327: Unicode 10328: Flight Recorder
> * 329: ChaCha20 & Poly1305 Cryptographic Algorithms
> * 330: Launch single-File Source-Code Programs
> * 331: Low-Overhead Heap Profiling
> * 332: Transport Layer Security (TLS)1.3
> * 333: ZGC: A Scalable Low-Latency Garbage Collector (Experimental)
> * 335: Deprecate the Nashorn JavaScript Engine
> * 336: Deprecate the Pack200 Tools and API
> 
> along with, of course, hundreds of smaller enhancements and countless bug fixes.
> 
> Thanks to everyone who contributed JDK 11, whether by creating features of enhancements, removing old features, fixing bugs, or downloading and testing the early-access builds.
>
> Onward, to JDK 12!

### ZGC: 可扩展的低延迟垃圾收集器
ZGC是一款号称可以保证每次GC的停顿时间不超过10MS的垃圾回收器，并且和当前的默认垃圾回收起G1相比，吞吐量下降不超过15%。
### psilon：什么事也不做的垃圾回收器
Java 11还加入了一个比较特殊的垃圾回收器——Epsilon，该垃圾收集器被称为“no-op”收集器，将处理内存分配而不实施任何实际的内存回收机制。 也就是说，这是一款不做垃圾回收的垃圾回收器。这个垃圾回收器看起来并没什么用，<u>主要可以用来进行性能测试、内存压力测试等，Epsilon GC可以作为度量其他垃圾回收器性能的对照组</u>。大神Martijn说，Epsilon GC至少能够帮助理解GC的接口，有助于成就一个更加模块化的JVM。
### 增强var用法
Java 10中增加了本地变量类型推断的特性，可以使用var来定义局部变量。尽管这一特性被很多人诟病，但是并不影响Java继续增强他的用法，在Java 11中，var可以用来作为Lambda表达式的局部变量声明。
### 移除Java EE和CORBA模块
早在发布Java SE 9的时候，Java就表示过，会在未来版本中将Java EE和CORBA模块移除，而这样举动终于在Java 11中实施。终于去除了Java EE和CORBA模块。
### HTTP客户端进一步升级
JDK 9 中就已对 HTTP Client API 进行标准化，然后通过JEP 110，在 JDK 10 中进行了更新。在本次的Java 11的更新列表中，由以JEP 321进行进一步升级。该API通过CompleteableFutures提供非阻塞请求和响应语义，可以联合使用以触发相应的动作。 JDK 11完全重写了该功能。现在，在用户层请求发布者和响应发布者与底层套接字之间追踪数据流更容易了，这降低了复杂性，并最大程度上提高了HTTP / 1和HTTP / 2之间的重用的可能性。