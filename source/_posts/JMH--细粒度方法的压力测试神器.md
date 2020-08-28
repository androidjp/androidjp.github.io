---
title: JMH--细粒度方法的压力测试神器
date: 2020-08-28 14:00:00
tags:
- jdk
categories:
- java
---

# 什么是基准测试？
基准测试（benchmark）：性能测试的一种。

<!-- more -->

## 定义
通过设计合理的测试方法，选用合适的测试工具和被测系统，实现对某个特定目标场景的某项性能指标进行定量的和可对比的测试。

## 特性
* **可重复性**：可进行重复性的测试，这样做有利于比较每次的测试结果，得到性能结果的长期变化趋势，为系统调优和上线前的容量规划做参考。
   > PS: 这种特质是为了满足基准测试的日常轮询需要。
* **可观测性**：通过全方位的监控（包括测试开始到结束，执行机、服务器、数据库），及时了解和分析测试过程发生了什么。
* **可展示性**：相关人员可以直观明了的了解测试结果（web界面、仪表盘、折线图树状图等形式）。
* **真实性**：测试的结果反映了客户体验到的真实的情况（真实准确的业务场景+与生产一致的配置+合理正确的测试方法）。
* **可执行性**：相关人员可以快速地进行测试验证修改调优（可定位可分析）。

## 前置条件

基准测试一定要在可控条件下进行。就是不要嘴上说着需求只是一天几次CRUD，上线才说要搞个淘宝亿万级并发。

才能得到相对准确的结果，为容量规划、缺陷定位、系统调优提供参考和依据。


# JMH简介
JMH is a Java harness for building, running, and analysing nano/micro/milli/macro benchmarks written in Java and other languages targetting the JVM
> 由简介可知，JMH不止能对Java语言做基准测试，还能对运行在JVM上的其他语言做基准测试。而且可以分析到纳秒级别。

# Maven库
```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.25.1</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.25.1</version>
</dependency>
```

# 推荐用法
官方推荐创建一个独立的Maven工程来运行JMH基准测试

## 一、用mvn命令创建工程
```
mvn archetype:generate 
-DinteractiveMode=false 
-DarchetypeGroupId=org.openjdk.jmh 
-DarchetypeArtifactId=jmh-java-benchmark-archetype 
-DgroupId=com.afei.jmh 
-DartifactId=jmh-test-demo 
-Dversion=1.0.0-SNAPSHOT
```
或者
```
mvn archetype:generate
```
然后一步步往下走，创建项目。

注意使用generate的maven项目，update一下依赖和编译jdk的版本：
```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <jmh.version>1.23</jmh.version>
    <javac.target>1.8</javac.target>
    <uberjar.name>benchmarks</uberjar.name>
</properties>
```

## 二、IDEA打开项目，写一个你想测的代码
如：
```java
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(1)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class MyBenchmark {

    @Benchmark
    public List<Integer> testMethod() {
        int cardCount = 54;
        List<Integer> cardList = new ArrayList<Integer>();
        for (int i=0; i<cardCount; i++){
            cardList.add(i);
        }
        // 洗牌算法
        Random random = new Random();
        for (int i=0; i<cardCount; i++) {
            int rand = random.nextInt(cardCount);
            Collections.swap(cardList, i, rand);
        }
        return cardList;
    }
}
```
```java
@Fork(1)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
public class Benchmark02 {

    static class Demo {
        int id;
        String name;
        public Demo(int id, String name) {
            this.id = id;
            this.name = name;
        }
    }

    static List<Demo> demoList;
    static {
        demoList = new ArrayList();
        for (int i = 0; i < 10000; i ++) {
            demoList.add(new Demo(i, "test"));
        }
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.SECONDS)
    public void testHashMapWithoutSize() {
        Map map = new HashMap();
        for (Demo demo : demoList) {
            map.put(demo.id, demo.name);
        }
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void testHashMap() {
        Map map = new HashMap((int)(demoList.size() / 0.75f) + 1);
        for (Demo demo : demoList) {
            map.put(demo.id, demo.name);
        }
    }
}
```

## 运行压测方式一：直接写一个main方法
```java
public static void main(String[] args) throws RunnerException {
    Options opt = new OptionsBuilder()
            .include(Benchmark02.class.getSimpleName())
            .include(MyBenchmark.class.getSimpleName())
            .forks(1)
            .build();

    new Runner(opt).run();
}
```
## 运行压测方式二：打包并运行jar包
### 1. 用mvn打包项目
```
mvn clean package
```
会看到`/target/benchmarks.jar`文件，具体打包得到的jar包文件名，是在`pom.xml`文件中配置的：
```xml
<configuration>
    <finalName>benchmarks.jar</finalName>
```

### 2. 执行jar
```
java -jar target/benchmarks.jar
```

# 运行结果分析
得到输出结果：
```
# VM invoker: C:\Program Files\Java\jre1.8.0_181\bin\java.exe
# VM options: <none>
# Warmup: 5 iterations, 1 s each
# Measurement: 5 iterations, 1 s each
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: com.afei.jmh.MyBenchmark.testMethod

# Run progress: 0.00% complete, ETA 00:00:10
# Fork: 1 of 1
# Warmup Iteration   1: 913.311 ns/op
# Warmup Iteration   2: 829.894 ns/op
# Warmup Iteration   3: 856.890 ns/op
# Warmup Iteration   4: 851.876 ns/op
# Warmup Iteration   5: 849.720 ns/op
Iteration   1: 863.683 ns/op
Iteration   2: 852.417 ns/op
Iteration   3: 856.067 ns/op
Iteration   4: 869.006 ns/op
Iteration   5: 878.699 ns/op


Result: 863.974 ±(99.9%) 40.309 ns/op [Average]
  Statistics: (min, avg, max) = (852.417, 863.974, 878.699), stdev = 10.468
  Confidence interval (99.9%): [823.665, 904.283]


# Run complete. Total time: 00:00:12

Benchmark                       Mode  Samples    Score  Score error  Units
c.a.j.MyBenchmark.testMethod    avgt        5  863.974       40.309  ns/op
```

## @Benchmarks
`@Benchmark`标签是用来标记测试方法的，只有被这个注解标记的话，该方法才会参与基准测试，但是有一个基本的原则就是被`@Benchmark`标记的方法必须是**public**的。

## @Warmup
参数解释：
* iterations：预热的次数。
* time：每次预热的时间。
* timeUnit：时间单位，默认是s。
* batchSize：批处理大小，每次操作调用几次方法。（后面用到）

```
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
```
表示预热5秒（迭代5次，每次1秒）。
对应输出日志：
```
# Warmup Iteration   1: 913.311 ns/op
# Warmup Iteration   2: 829.894 ns/op
# Warmup Iteration   3: 856.890 ns/op
# Warmup Iteration   4: 851.876 ns/op
# Warmup Iteration   5: 849.720 ns/op
```
> 预热？为什么要预热？因为 JVM 的 JIT 机制的存在，如果某个函数被调用多次之后，JVM 会尝试将其编译成为机器码从而提高执行速度。为了让 benchmark 的结果更加接近真实情况就需要进行预热。

## @Measurement
例子：
```
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
```
表示循环运行5次，总计5秒时间。

## @Fork
这个注解表示fork多少个线程运行基准测试：
* `@Fork(1)`：那么就是一个线程，这时候就是同步模式。

## @BenchmarkMode && @OutputTimeUnit
基准测试模式申明为：
* `@BenchmarkMode(Mode.AverageTime)`
  * `Mode`有以下几个options：
    * `Mode.Throughput`：每毫秒的吞吐量（即每毫秒多少次操作）
    * `Mode.AverageTime`：平均耗时
    * `Mode.SampleTime`：抽样检测
    * `Mode.SingleShotTime`：检测一次调用
    * `Mode.All`：运行所有的检测模式
* `@OutputTimeUnit(TimeUnit.NANOSECONDS)`（可选基准测试模式通过枚举Mode得到）

可以这样同时多个维度对目标方法进行测量：`@BenchmarkMode({Mode.Throughput, Mode.AverageTime, Mode.SampleTime, Mode.SingleShotTime})`

笔者的示例是AverageTime，即表示每次操作需要的平均时间，而OutputTimeUnit申明为纳秒，所以基准测试单位是ns/op，即每次操作的纳秒单位平均时间。基准测试结果如下：
```
Result: 863.974 ±(99.9%) 40.309 ns/op [Average]
  Statistics: (min, avg, max) = (852.417, 863.974, 878.699), stdev = 10.468
  Confidence interval (99.9%): [823.665, 904.283]
```
最后一段结果如下，重点关注**Mean**和**Units**两个字段，组合起来就是 **863.974ns/op**，即每次操作耗时863.974纳秒：

对于 **Mean error**：表示误差，或者波动。


## @State
在很多时候我们需要维护一些状态内容，比如在多线程的时候我们会维护一个共享的状态，这个状态值可能会在每隔线程中都一样，也有可能是每个线程都有自己的状态，JMH为我们提供了状态的支持。该注解只能用来标注在类上，因为类作为一个属性的载体。 @State的状态值主要有以下几种：
* `Scope.Benchmark` 该状态的意思是会在所有的Benchmark的工作线程中共享变量内容。
* `Scope.Group` 同一个Group的线程可以享有同样的变量
* `Scope.Thread` 每隔线程都享有一份变量的副本，线程之间对于变量的修改不会相互影响。
下面看两个常见的@State的写法：

1.直接在内部类中使用@State作为“PropertyHolder”
```java
public class JMHSample_03_States {

    @State(Scope.Benchmark)
    public static class BenchmarkState {
        volatile double x = Math.PI;
    }

    @State(Scope.Thread)
    public static class ThreadState {
        volatile double x = Math.PI;
    }

    @Benchmark
    public void measureUnshared(ThreadState state) {
        state.x++;
    }

    @Benchmark
    public void measureShared(BenchmarkState state) {
        state.x++;
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(JMHSample_03_States.class.getSimpleName())
                .threads(4)
                .forks(1)
                .build();

        new Runner(opt).run();
    }
}
```
2.在Main类中直接使用@State作为注解，是Main类直接成为“PropertyHolder”
```java
@State(Scope.Thread)
public class JMHSample_04_DefaultState {
    double x = Math.PI;

    @Benchmark
    public void measure() {
        x++;
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(JMHSample_04_DefaultState.class.getSimpleName())
                .forks(1)
                .build();

        new Runner(opt).run();
    }
}
```
我们试想以下@State的含义，它主要是方便框架来控制变量的过程逻辑，通过@State标示的类都被用作属性的容器，然后框架可以通过自己的控制来配置不同级别的隔离情况。被@Benchmark标注的方法可以有参数，但是参数必须是被@State注解的，就是为了要控制参数的隔离。
但是有些情况下我们需要对参数进行一些初始化或者释放的操作，就像Spring提供的一些init和destory方法一样，JHM也提供有这样的钩子：
* `@Setup` 必须标示在@State注解的类内部，表示初始化操作
* `@TearDown` 必须表示在@State注解的类内部，表示销毁操作

初始化和销毁的动作都只会执行一次。
```java
@State(Scope.Thread)
public class JMHSample_05_StateFixtures {
    double x;

    @Setup
    public void prepare() {
        x = Math.PI;
    }

    @TearDown
    public void check() {
        assert x > Math.PI : "Nothing changed?";
    }

    @Benchmark
    public void measureRight() {
        x++;
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(JMHSample_05_StateFixtures.class.getSimpleName())
                .forks(1)
                .jvmArgs("-ea")
                .build();

        new Runner(opt).run();
    }
}
```
虽然我们可以执行初始化和销毁的动作，但是总是感觉还缺点啥？对，就是初始化的粒度。因为基准测试往往会执行多次，那么能不能保证每次执行方法的时候都初始化一次变量呢？ @Setup和@TearDown提供了以下三种纬度的控制：
* `Level.Trial` 只会在个基础测试的前后执行。包括Warmup和Measurement阶段，一共只会执行一次。
* `Level.Iteration` 每次执行记住测试方法的时候都会执行，如果Warmup和Measurement都配置了2次执行的话，那么@Setup和@TearDown配置的方法的执行次数就4次。
* `Level.Invocation` 每个方法执行的前后执行（一般不推荐这么用）

## @Param
在很多情况下，我们需要测试不同的参数的不同结果，但是测试的了逻辑又都是一样的，因此如果我们编写镀铬benchmark的话会造成逻辑的冗余，幸好JMH提供了`@Param`参数来帮助我们处理这个事情，被`@Param`注解标示的参数组会一次被benchmark消费到。
```java
@State(Scope.Benchmark)
public class ParamTest {

    @Param({"1", "2", "3"})
    int testNum;

    @Benchmark
    public String test() {
        return String.valueOf(testNum);
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(ParamTest.class.getSimpleName())
                .forks(1)
                .build();

        new Runner(opt).run();
    }
}
```

## @Threads
测试线程的数量，可以配置在方法或者类上，代表执行测试的线程数量。
> 通常看到这里我们会比较迷惑Iteration和Invocation区别，我们在配置Warmup的时候默认的时间是的1s，即1s的执行作为一个Iteration，假设每次方法的执行是100ms的话，那么1个Iteration就代表10个Invocation。

# JMH进阶

## 不要编写无用代码
因为现代的编译器非常聪明，如果我们在代码使用了没有用处的变量的话，就容易被编译器优化掉，这就会导致实际的测量结果可能不准确，因为我们要在测量的方法中避免使用void方法，然后记得在测量的结束位置返回结果。这么做的目的很明确，就是为了与编译器斗智斗勇，让编译器不要改变这段代码执行的初衷。

## Blackhole介绍
Blackhole会消费传进来的值，不提供任何信息来确定这些值是否在之后被实际使用。
Blackhole处理的事情主要有以下几种：
* 死代码消除：入参应该在每次都被用到，因此编译器就不会把这些参数优化为常量或者在计算的过程中对他们进行其他优化。
* 处理内存壁：我们需要尽可能减少写的量，因为它会干扰缓存，污染写缓冲区等。
这很可能导致过早地撞到内存壁

我们在上面说到需要消除无用代码，那么其中一种方式就是通过Blackhole，我们可以用Blackhole来消费这些返回的结果。
```java
// 1:返回测试结果，防止编译器优化
@Benchmark
public double measureRight_1() {
    return Math.log(x1) + Math.log(x2);
}

// 2.通过Blackhole消费中间结果，防止编译器优化
@Benchmark
public void measureRight_2(Blackhole bh) {
    bh.consume(Math.log(x1));
    bh.consume(Math.log(x2));
}
```

## 循环处理
我们虽然可以在Benchmark中定义循环逻辑，但是这么做其实是不合适的，因为编译器可能会将我们的循环进行展开或者做一些其他方面的循环优化，所以JHM建议我们不要在Beanchmark中使用循环，如果我们需要处理循环逻辑了，可以结合@BenchmarkMode(Mode.SingleShotTime)和@Measurement(batchSize = N)来达到同样的效果.
```java
@State(Scope.Thread)
public class JMHSample_26_BatchSize {

    List<String> list = new LinkedList<>();
    
    // 每个iteration中做5000次Invocation
    @Benchmark
    @Warmup(iterations = 5, batchSize = 5000)
    @Measurement(iterations = 5, batchSize = 5000)
    @BenchmarkMode(Mode.SingleShotTime)
    public List<String> measureRight() {
        list.add(list.size() / 2, "something");
        return list;
    }

    @Setup(Level.Iteration)
    public void setup(){
        list.clear();
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(JMHSample_26_BatchSize.class.getSimpleName())
                .forks(1)
                .build();

        new Runner(opt).run();
    }

}
```
## 方法内联
方法内联：如果JVM监测到一些小方法被频繁的执行，它会把方法的调用替换成方法体本身。比如说下面这个：
```java
private int add4(int x1, int x2, int x3, int x4) {
    return add2(x1, x2) + add2(x3, x4);
}

private int add2(int x1, int x2) {
    return x1 + x2;
}
```
运行一段时间后JVM会把add2方法去掉，并把你的代码翻译成：
```java
private int add4(int x1, int x2, int x3, int x4) {
    return x1 + x2 + x3 + x4;
}
```
JMH提供了CompilerControl注解来控制方法内联，但是实际上我感觉比较有用的就是两个了：
* CompilerControl.Mode.DONT_INLINE：强制限制不能使用内联
* CompilerControl.Mode.INLINE：强制使用内联

看一下官方提供的例子把：
```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class JMHSample_16_CompilerControl {

    public void target_blank() {
        
    }

    @CompilerControl(CompilerControl.Mode.DONT_INLINE)
    public void target_dontInline() {
        
    }

    @CompilerControl(CompilerControl.Mode.INLINE)
    public void target_inline() {
        
    }
  
    @Benchmark
    public void baseline() {
        
    }

    @Benchmark
    public void dontinline() {
        target_dontInline();
    }

    @Benchmark
    public void inline() {
        target_inline();
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(JMHSample_16_CompilerControl.class.getSimpleName())
                .warmupIterations(0)
                .measurementIterations(3)
                .forks(1)
                .build();

        new Runner(opt).run();
    }
}

======================================执行结果==============================
Benchmark                                Mode  Cnt  Score   Error  Units
JMHSample_16_CompilerControl.baseline    avgt    3  0.896 ± 3.426  ns/op
JMHSample_16_CompilerControl.dontinline  avgt    3  0.344 ± 0.126  ns/op
JMHSample_16_CompilerControl.inline      avgt    3  0.391 ± 2.622  ns/op
======================================执行结果==============================
```

# JMH和jMeter的不同
JMH和jMeter的使用场景还是有很大的不同的，jMeter更多的是对rest api进行压测，而JMH关注的粒度更细，它更多的是发现某块性能槽点代码，然后对优化方案进行基准测试对比。比如json序列化方案对比，bean copy方案对比，文中提高的洗牌算法对比等。


# 参考文章
* https://www.jianshu.com/p/0da2988b9846
* https://www.jianshu.com/p/23abfe3251ca
* https://juejin.im/post/5c471d3ff265da61427433b3