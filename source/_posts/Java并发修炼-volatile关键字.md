---
title: Java并发修炼--volatile关键字
date: 2019-10-12 09:51:43
tags:
- java
- 多线程
categories:
- java
---
# 如何理解volatile关键字
被 `volatile` 修饰的共享变量，会满足以下两个性质：
* 保证了不同线程对该变量操作的内存可见性;
* 禁止指令重排序

# 什么是内存可见性
我们先回顾以下JMM（Java内存模型）。

Java虚拟机规范试图定义一种Java内存模型（JMM）,来屏蔽掉各种硬件和操作系统的内存访问差异，让Java程序在各种平台上都能达到一致的内存访问效果。简单来说，由于CPU执行指令的速度是很快的，但是内存访问的速度就慢了很多，相差的不是一个数量级（就速度而言：CPU高速缓存 >> 内存 >> 磁盘），所以搞处理器的那群大佬们又在CPU里加了好几层高速缓存。

JMM规定所有变量都是存在主存中的，类似于上面提到的普通内存，每个线程又包含自己的工作内存，方便理解就可以看成CPU上的寄存器或者高速缓存。所以线程的操作都是以工作内存为主，它们只能访问自己的工作内存，且工作前后都要把值在同步回主内存。
> 相当于：我和妈妈分别在各自手机中缓存我的vlog，虽然vlog在网络上是大家共享的资源，但是还是缓存在各自的手机上，想看不用等。看完后我和妈妈都可以修改我的vlog，修改好可以立即上传到网络，也可以等一会再上传。

![loAC2d.png](https://s2.ax1x.com/2020/01/12/loAC2d.png)

线程执行时：
1. 首先会从主存中read变量值；
2. 再将变量值load到工作内存中的副本中；
3. 然后再传给处理器执行；
4. 在执行完毕后再给工作内存中的副本赋值；
5. 随后工作内存再把值传回给主内存，此时，主存更新

看一个例子：
```
i = i = 1;
```
假设i初值为0，当只有一个线程执行它时，结果肯定得到1，当两个线程执行时，会得到结果2吗？这倒不一定了。可能存在这种情况：

线程A | 线程B
 --- | ---
load i from 主存（i = 0）|
i+1（i = 1）|
| load i from 主存（i = 0）
| i+1（i = 1）
save i to 主存（i = 1）|
| save i to 主存（i = 1）

这样一来，就可能出现 “少加” 的情况，如果最后的写回生效的慢，你再读取i的值，都可能是0，也就是**缓存不一致问题**。

JMM主要就是围绕着如何在并发过程中如何处理**原子性**、**可见性**和**有序性**这3个特征来建立的，通过解决这三个问题，可以解除缓存不一致的问题。而`volatile`跟**可见性**和**有序性**都有关。
> 注意，这里说的几个特征，是说的JMM要解决和保证的特征，类似的，事务也有几大特性：ACID
> * Atomicity 原子性
> * Consistency 一致性
> * Isolcation 隔离性
> * Durability 持久性
>
> 这两者所说的，也就只有 “原子性”可以相提并论。所以，不要搞混！

## 1. 原子性（Atomicty）
Java中，对基本数据类型的读取和赋值操作是原子性操作，所谓原子性操作就是指这些操作是不可中断的，要做一定做完，要么就没有执行。如：
```
i = 2;     // 原子性操作
j = i;     // 2步操作：先读取i的值，然后赋值给j
i++;       // 3步操作：先读取i的值，然后加1，最后写回主内存
i = i + 1; // 3步操作：同上
```
所以，严格来说，只有`i = 2;` 满足原子性。
> 有个例外是，虚拟机规范中允许对64位数据类型(long和double)，分为2次32为的操作来处理，但是最新JDK实现还是实现了原子操作的。

JMM只实现了基本的原子性，像上面`i++`那样的操作，必须借助于`synchronized`和`Lock`来保证整块代码的原子性了。线程在释放锁之前，必然会把 `i` 的值刷回到主存的。

## 2. 可见性（Visibility）
说到可见性，Java就是利用volatile来提供可见性的。
当一个变量被volatile修饰时，那么对它的修改会立刻刷新到主存，当其它线程需要读取该变量时，会去内存中读取新值。而普通变量则不能保证这一点。
其实通过`synchronized`和`Lock`也能够保证可见性，线程在释放锁之前，会把共享变量值都刷回主存，但是`synchronized`和`Lock`的开销都更大。

综上所述，我们得有一个认识：`volatile`和`synchronized`所说的“可见性”，实际上都是基于锁，或者一种约定，这个在后面会说。

## 3. 有序性（Ordering）
JMM是允许编译器和处理器对指令重排序的，但是规定了**as-if-serial**语义，即不管怎么重排序，程序的执行结果不能改变。比如下面的程序段：
```
double pi = 3.14;    //A
double r = 1;        //B
double s= pi * r * r;//C
```
以上代码，可以是 `A -> B -> C`，也可以是 `B -> A -> C`，因为A和B相互不依赖，而C则依赖A和B，所以，A和B随便怎么换，但是C不能排到A或B前面。**JMM保证了重排序不会影响到单线程的执行，但是在多线程中却容易出问题**。

如:
```
int a = 0;
bool flag = false;

public void write() {
    a = 2;              //1
    flag = true;        //2
}

public void multiply() {
    if (flag) {         //3
        int ret = a * a;//4
    }   
}
```
以上代码，如果单线程执行`write() -> multiply()` 是完全没问题的。但是，如果线程A执行`write`，线程B执行`multiply`，那就可能出现**多线程时指令重排序问题（无法保证多线程时的结果最终一致性）**

 线程A | 线程B
--- | ---
 flag = true;|
| if(flag)
| int ret = a * a;
 a = 2;|

这时候可以为flag加上volatile关键字，禁止重排序，可以确保程序的“有序性”，也可以上重量级的synchronized和Lock来保证有序性,它们能保证那一块区域里的代码都是一次性执行完毕的。

# JMM具备一些先天的有序性
JMM具备一些先天的有序性,即不需要通过任何手段就可以保证的有序性，通常称为happens-before原则。`<<JSR-133：Java Memory Model and Thread Specification>>`定义了如下happens-before规则：
> 1. **程序顺序规则**： 一个线程中的每个操作，happens-before于该线程中的任意后续操作
> 2. **监视器锁规则**：对一个线程的解锁，happens-before于随后对这个线程的加锁
> 3. **volatile变量规则**： 对一个volatile域的**写**，happens-before于后续对这个volatile域的**读**
> 4. **传递性**：如果A happens-before B ,且 B happens-before C, 那么 A happens-before C
> 5. **start()规则**： 如果线程A执行操作ThreadB_start()(启动线程B) ,  那么A线程的ThreadB_start()happens-before 于B中的任意操作
> 6. **join()原则**： 如果A执行ThreadB.join()并且成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。
> 7. **interrupt()原则**： 对线程interrupt()方法的调用先行发生于被中断线程代码检测到中断事件的发生，可以通过Thread.interrupted()方法检测是否有中断发生
> 8. **finalize()原则**：一个对象的初始化完成先行发生于它的finalize()方法的开始

第1条规则---程序顺序规则，是说在一个线程里，所有的操作都是按顺序的，**但是**在JMM里其实只要执行结果一样，是允许重排序的，这边的happens-before强调的重点也是单线程执行结果的正确性，但是无法保证多线程也是如此。

第2条规则监视器规则其实也好理解，就是在加锁之前，确定这个锁之前已经被释放了，才能继续加锁。

第3条规则，就适用到所讨论的volatile，如果一个线程先去写一个变量，另外一个线程再去读，那么写入操作一定在读操作之前。

第4条规则，就是happens-before的传递性。
后面几条就不再一一赘述了。

# volatile关键字如何保证可见性和有序性
那就要重提volatile变量规则： 对一个volatile域的写，happens-before于后续对这个volatile域的读。
这条再拎出来说，其实就是如果一个变量声明成是volatile的，那么当我读变量时，总是能读到它的最新值，这里最新值是指不管其它哪个线程对该变量做了写操作，都会立刻被更新到主存里，我也能从主存里读到这个刚写入的值。也就是说volatile关键字可以保证可见性以及有序性。

如果这样不能理解，或者我们重看文章开头的图：
![loAC2d.png](https://s2.ax1x.com/2020/01/12/loAC2d.png)
volatile的特殊性在于：
1. 操作 `use` 之前必须先执行 `read`和`load`操作。
2. 操作 `assign` 之后必须执行 `store`和`write`操作。

由特性性保证了read、load和use的操作连续性，assign、store和write的操作连续性，从而达到工作内存读取前必须刷新主存最新值；工作内存写入后必须同步到主存中。读取的连续性和写入的连续性，看上去像线程直接操作了主存，又因为主存实际是多线程可见的，于是可见性就体现了出来。
>  扩展：
上图的lock和unlock操作并不直接开放给用户使用，而是提供给像Synchronize关键字指定monitorenter和monitorexit隐式使用。关于Synchronize的监听器锁monitor，javac编译后会在作用的方法前后增加monitorenter和monitorexit指令，详细的可以查看Synchronize原理。

继续看回上面提到的case代码：
```
int a = 0;
bool flag = false;

public void write() {
   a = 2;              //1
   flag = true;        //2
}

public void multiply() {
   if (flag) {         //3
       int ret = a * a;//4
   }  
}
```
这种正常在多线程情况下，除了指令重排序的风险，也会有线程执行先后顺序的问题。也就是说，即使`1 --> 2` 不会变成`2 --> 1`，`2`和`3`的执行顺序也不一定就是`2-->3`，因为是多个线程在跑。

我们让`flag`加上`volatile`关键字：
```
int a = 0;
volatile bool flag = false;

public void write() {
   a = 2;              //1
   flag = true;        //2
}

public void multiply() {
   if (flag) {         //3
       int ret = a * a;//4
   }
}
```
那么线程A先执行`write`,线程B再执行`multiply`。根据happens-before原则，这个过程会满足以下3类规则：
1. 程序顺序规则：1 happens-before 2; 3 happens-before 4; (volatile限制了指令重排序，所以1 在 2 之前执行)
2. volatile规则：2 happens-before 3
3. 传递性规则：1 happens-before 4

从内存语义上来看:
* **当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存**
* **当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效，线程接下来将从主内存中读取共享变量**

# volatile关键字能否保证原子性
**不能**。

硬要说能，就是得看角度了。前面我们说单个变量的读/写操作，具有原子性，这个性质在Java里头是基本规则。

```
public class Test {
    public volatile int inc = 0;
 
    public void increase() {
        inc++;
    }
 
    public static void main(String[] args) {
        final Test test = new Test();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
 
        Thread.sleep(6000L);
        System.out.println(test.inc);
    }
}
```
你很容易得到一个小于10000的值。

有人可能会说volatile不是保证了可见性啊，一个线程对inc的修改，另外一个线程应该立刻看到啊！可是这里的操作inc++是个复合操作啊，包括读取inc的值，对其自增，然后再写回主存。

假设线程A，读取了inc的值为10，这时候被阻塞了，因为没有对变量进行修改，触发不了volatile规则。

线程B此时也读读inc的值，主存里inc的值依旧为10，做自增，然后立刻就被写回主存了，为11。

此时又轮到线程A执行，由于工作内存里保存的是10，所以继续做自增，再写回主存，11又被写了一遍。所以虽然两个线程执行了两次increase()，结果却只加了一次。

有人说，`volatile不是会使缓存行无效的吗？`但是这里线程A读取时，线程B还没有进行修改操作，所以并没有修改inc值，并不会触发volatile功能，所以线程B读取的时候，还是读的10，因为B不知道A要修改inc。

又有人说，线程B将11写回主存，`不会把线程A的缓存行设为无效吗？`但是线程A的读取操作已经做过了啊，只有在做读取操作时，发现自己缓存行无效，才会去读主存的值，所以这里线程A只能继续做自增了。

因此，要想保证原子性，只能借助于`synchronized`,`Lock`以及并发包下的atomic的原子操作类了，即对基本数据类型的 自增（加1操作），自减（减1操作）、以及加法操作（加一个数），减法操作（减一个数）进行了封装，保证这些操作是原子性操作。

# volatile底层实现机制
通过在volatile变量的操作前后插入**内存屏障**的方式，控制后置指令排序无法排到屏障之前，并且使得CPU的Cache写入内存，写入动作也会触发别的CPU或内核无效化其Cache。


如果把加入volatile关键字的代码和未加入volatile关键字的代码都生成汇编代码，会发现加入volatile关键字的代码会多出一个lock前缀指令。

lock前缀指令实际相当于一个内存屏障，内存屏障提供了以下功能：
1. 重排序时不能把后面的指令重排序到内存屏障之前的位置；
2. 使得本CPU的Cache写入内存；
3. 写入动作也会引起别的CPU或者别的内核无效化其Cache，相当于让新写入的值对别的线程可见。

# volatile 与 synchronized 的关系
## 相同点
volatile 与 synchronized 都属于关键字。

volatile 与 synchronized 都能保证可见性和有序性。

## 区别
只是分别对应的有序性规则不同，且实现方式也不同：
* volatile角度的有序性偏向于对目标对象的“写”操作必须保证在“读”操作之前，并通过内存屏障去保证目标对象相关代码的执行顺序。
* synchronized 角度来看并不是说代码连执行顺序都不能变，而是想要保证最终结果一致性，然后，相关可优化的代码还是可以被重排序的。并通过加锁机制，并不是锁定代码执行顺序，而是锁定“厕所门”，同一时间片下，只能也给一个线程执行这一段被“锁”住的代码。

> synchronized是无法禁止指令重排和处理器优化的。那么他是如何保证的有序性呢？
> 
> 答： 这和as-if-serial语义有关。as-if-serial语义的意思指：不管怎么重排序，单线程程序的执行结果都不能被改变。编译器和处理器无论如何优化，都必须遵守as-if-serial语义。简单说就是，as-if-serial语义保证了单线程中，不管指令怎么重排，最终的执行结果是不能被改变的。

volatile关键字是无法保证原子性的，而synchronized通过monitorenter和monitorexit两个指令，可以保证被synchronized修饰的代码在同一时间只能被一个线程访问，即可保证不会出现CPU时间片在多个线程间切换，即可保证原子性。

## synchronized 比 volatile 重
synchronized其实是一种加锁机制，那么既然是锁，天然就具备以下几个缺点：
1. 有性能损耗
   > 虽然在JDK 1.6中对synchronized做了很多优化，如如适应性自旋、锁消除、锁粗化、轻量级锁和偏向锁等，但是他毕竟还是一种锁。
   >
   > 以上这几种优化，都是尽量想办法避免对Monitor进行加锁，但是，并不是所有情况都可以优化的，况且就算是经过优化，优化的过程也是有一定的耗时的。
   >
   > 所以，无论是使用同步方法还是同步代码块，在同步操作之前还是要进行加锁，同步操作之后需要进行解锁，这个加锁、解锁的过程是要有性能损耗的。
   >
   > 关于二者的性能对比，由于虚拟机对锁实行的许多消除和优化，使得我们很难量化这两者之间的性能差距，但是我们可以确定的一个基本原则是：volatile变量的读操作的性能小号普通变量几乎无差别，但是写操作由于需要插入内存屏障所以会慢一些，即便如此，volatile在大多数场景下也比锁的开销要低。

2. 产生阻塞
   > 无论是同步方法还是同步代码块，无论是ACC_SYNCHRONIZED还是monitorenter、monitorexit都是基于Monitor实现的。
   >
   > 基于Monitor对象，当多个线程同时访问一段同步代码时，首先会进入Entry Set，当有一个线程获取到对象的锁之后，才能进行The Owner区域，其他线程还会继续在Entry Set等待。并且当某个线程调用了wait方法后，会释放锁并进入Wait Set等待。

所以，synchronize实现的锁本质上是一种阻塞锁，也就是说多个线程要排队访问同一个共享对象。

而volatile是Java虚拟机提供的一种轻量级同步机制，他是基于内存屏障实现的。说到底，他并不是锁，所以他不会有synchronized带来的阻塞和性能损耗的问题。

# 为什么DSL的单例模式中，instance要有volatile？
看看以下这个单例，一个典型的不完善的DSL（双重锁检查）单例：
```
public class Singleton {
    private static Singleton instance = null;

    privat Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized(Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
    }
}
```
为什么说它不完善呢？ 虽然在`synchronized`同步代码块内，能够保证同一时刻只会有一个线程执行，但是，`instance = new Singleton();` 始终是一个复合操作，相当于 2个步骤：
1. JVM为对象分配一块内存M；
2. 在内存M上为对象进行初始化`new Singleton()`；
3. 将内存M的地址复制给instance变量：`instance = addr`

所以可能会出现 `1 --> 3 --> 2`这样的指令重排序情况，最终拿到了一个不完整的singleton对象，极有可能发生NPE异常，所以，加上`volatile`：

```
public class Singleton {
    private volatile static Singleton instance = null;

    privat Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized(Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
    }
}
```