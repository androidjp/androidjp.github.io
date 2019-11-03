* https://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247489736&idx=1&sn=4ff3c1c19db85a100e2fd62dd4d60ea3&chksm=ebd627e4dca1aef2246ba025b188cd67f556f109f482cf249d5ddce7a7b13caefd3e30de7eae&mpshare=1&scene=1&srcid=0922Bv23BP36L0W2G21UF77i&sharer_sharetime=1569150055237&sharer_shareid=ba40c6ff48492b2ca4764de8dad7545e&key=f73120b8a2b526d17c66c7a8c063ef6612127d9b9a3a3ecf95e0d2983227b271250a180e1beb830174c71e1aa69f78d37919c424740b789dddeed6d1ecff90a1174fc400b90ea30355f3aed71ce4e83c&ascene=1&uin=MjU1MDY2OTI0Mw%3D%3D&devicetype=Windows+10&version=62060834&lang=zh_CN&pass_ticket=p90ytQ4OStkEjUHpogr%2BTyU3KopX4YL8mxWRXkqNJNqMfQBJB4ssqfv7r%2FyaOr%2F0

* https://mp.weixin.qq.com/s?__biz=MzU4ODI1MjA3NQ==&mid=2247484500&idx=2&sn=a8751fcdd006850f03b4c663f975a02e&chksm=fdded290caa95b86178c0e022897a8b8de92434c3cab8536e1f6d491c4677fd1fdcbd239b8be&scene=0&xtrack=1&key=19f491736d4c878431f6df559c9474e31df3c98d353aacb47d4538c9d593177dca9855c9ef3f8760f098fa7a75f9b4b7261e15fe4f55d155dfdbb477b1f64db337c6854f32b9e4d62207905400257330&ascene=1&uin=MjU1MDY2OTI0Mw%3D%3D&devicetype=Windows+10&version=62070152&lang=zh_CN&pass_ticket=Er%2Fgq6T8yTJvDBNNHY7j8MVcyWo36NOvn5ldPuSaVwUPGvOdRnrfAhIZptQTvfdi

* https://juejin.im/post/5dba72c96fb9a02047526331?utm_source=gold_browser_extension


# Java局部变量用final修饰，到底有什么用？
## 场景一：内部类中使用到局部变量
除了内部类引用外部类方法的某个局部变量时，需要给这个局部变量声明`final`的场景：
```
class Outer{
    
    Object obj;
    public void outerMethod() {
        
        //局部变量
        int x = 5;
        //定义在方法中的内部类称为局部内部类
        class Inner{

            public String toString() {
                System.out.println(x);//访问了局部变量x
  　　　　　　　　　　return null;　
            }
        }
        //创建内部类实例
        Inner in = new Inner();
        in.toString();
        //将内部类实例的引用赋值给obj
        obj = in;
    }
}
public class HelloDemo {
    
    public static void main(String[] argr) {
        Outer out = new Outer();
        out.outerMethod();
    　　
　　}
}
```
如上面的第7行代码所示，变量x没有被声明为final，如果是这样的话，当执行完第26行的outMethod()方法后，outMethod()方法将出栈，出栈后outMethod()方法里面定义的所有变量（x 和 in）都死亡了，（但是此时内部类的对象还活着，直到它不再被使用才会被回收，也就是说此内部类对象的生命周期比局部变量的生命周期长），并且在变量 in 死亡之前，in 的值赋值给了成员变量obj（第19行代码），这时obj 指向了内部类的对象，如果此时在27行执行一条代码： out.obj.toString();那么这条代码将会访问到局部变量 x，但是此时 x 已经死亡了，内部类对象已经访问不到 x 了，因此这是相互矛盾的。所以上面的代码实际上并不能编译通过。（在jdk8.0能编译通过，那是因为它检测到局部内部类访问了 x，会默认给 x 的前面加上隐式的final，如果在第8行加上一句代码：x = 4；编译器将会报错，因为final不允许 x 的值改变）

如果局部变量 x 被声明为final后(在第七行的int前加上final)，x 就代表了一个常量，那么第15行的代码实际上就变成了 System.out.println(5);，这时内部类相当于访问了一个数字5。这是没任何问题的，因此局部内部类访问它所在方法的局部变量时，要求该局部变量必须声明为final。究其根本原因，是局部内部类对象的生命周期比局部变量的生命周期长。


此场景中，变量`x`加`final`修饰的原因：
1. 保证局部变量在匿名内部类内外都不会被修改。
因为匿名内部类内部，实际上是复制了一份局部变量，然后在匿名内部类中使用。如果不设置为final，局部变量在外部被修改，会导致与匿名内部类之内的副本不一致，逻辑上说不通。所以Java虚拟机这么设计，强制设置局部变量为final，语义上保持一致性。
2. 2. 局部变量的生命周期：当该方法被调用时，该方法中的局部变量在栈中被创建，当方法调用结束时，退栈，这些局部变量全部死亡。而内部类对象生命周期与其它类对象一样：自创建一个匿名内部类对象，系统为该对象分配内存，直到没有引用变量指向分配给该对象的内存，它才有可能会死亡（被JVM垃圾回收）。所以完全可能出现的一种情况是：成员方法已调用结束，局部变量已死亡，但匿名内部类的对象仍然活着。
3. 如果匿名内部类的对象访问了同一个方法中的局部变量，就要求只要匿名内部类对象还活着，那么栈中的那些它要所访问的局部变量就不能“死亡”。
4. 解决方法：匿名内部类对象可以访问同一个方法中被定义为final类型的局部变量。定义为final后，编译器会把匿名内部类对象要访问的所有final类型局部变量，都拷贝一份作为该对象的成员变量。这样，即使栈中局部变量已经死亡，匿名内部类对象照样可以拿到该局部变量的值，因为它自己拷贝了一份，且与原局部变量的值始终保持一致（final类型不可变）。




之外，我们还经常看到许多代码里头，方法里的局部变量都用`final`进行修饰，为什么呢？这样做有什么好处吗？

《并发编程实战》一书中的【3.4.1 Final域】提到

正如“除非需要更高的可见性，否则应将所有的域都声明为私有域”是一个良好的编程习惯，
“除非需要某个域是可变的，否则应将其声明为final域”也是一个良好的编程习惯。

参考了文章：
* [](https://blog.csdn.net/gdscp/article/details/72901063)
* [](https://www.cnblogs.com/jixp/articles/6752132.html)

发现final的好处主要有：
* 被final修饰的基本数据类型不可变，引用数据类型地址不可被修改；
* 访问final变量的速度快；


访问final变量的速度快这个结论有什么依据吗？

我所了解的，JVM会对final变量的访问会禁止重排序优化，如果不使用final,访问变量时会进行重排序优化从而提高性能，但是在多线程情况下，重排序可能会造成线程安全问题，所以使用final修饰共享变量会避免这种重排序。专业的话讲就是防止变量从构造方法中逸出。另外final修饰的类变量虽然是共享的，但不可变能保证他的线程安全。

不过用final修饰方法和类确实能提高性能，如果没有final修饰，那么该类可能会被继承，JVM需要为继承做一些准备，如果有final修饰，相当于告诉JVM，该类或方法是不会被继承的，所以JVM会省掉哪些准备的时间。

[参考回答](https://www.zhihu.com/question/21762917)

最重要部分是，用final修饰局部变量，编译器会进行常量折叠。

阿里开发手册上面对final的用法推荐：
【推荐】 final可以声明类、成员变量、方法、以及本地变量,下列情况使用 final关键字:
1)不允许被继承的类,如:5xing笑
2)不允许修改引用的域对象,如:POJO类的域变量。
3)不允许被重写的方法,如:POJO类的 setter方法。
4)不允许运行过程中重新赋值的局部变量。
5)避免上下文重复使用一个变量,使用 final描述可以强制重新定义一个变量,方便更好
地进行重构。

使用final可以使你的代码意图更加明确，将变量标记为final告诉读者该变量永远不会在分配时发生变化。这在阅读代码时非常重要