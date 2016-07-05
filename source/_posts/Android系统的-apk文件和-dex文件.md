---
title: Android系统的.apk文件和.dex文件
date: 2016-06-29 14:18:24
categories:
- Android
tags:
- Android
- apk相关
---

> 给大家简单介绍下 apk 和 dex 两种文件的概念以及他们的联系。

<!--more-->

## .apk文件

1. 概念
&emsp;&emsp;apk，全称Android Package，即Android安装包。
（通过apk文件直接传到Android模拟器或Android手机中执行即可安装。）
apk文件本质上是jar或zip文件

2. apk文件目录结构

![](http://i1.piimg.com/567571/f2e012e7f1fa25c5.png)

    * META-INF -- Jar文件
    * res -- 资源文件
    * AndroidManifest -- 应用全局配置文件
    * .dex -- Dalvik虚拟机字节码文件
    * resource.arsc -- 编译后的二进制资源文件

3. apk被调用的过程
<p>
Android在运行程序时首先需要解压apk文件，然后获取编译后的androidmanifest.xml文件中配置信息，执行dex程序。</p>


## .dex文件
1. 概念
<p>
Dex是Dalvik VM executes的全称，即Android Dalvik执行程序，本身是将app的除了界面通常执行时都进行优化 。优化后的文件大小会有所增加。 优化发生的时机有两个：对于预置应用，可以在系统编译后，生成优化文件，以ODEX结尾。这样在发布时除APK文件（不包含DEX）以外，还有一个相应的Android DEX文件；对于非预置应用，包含在APK文件里的DEX文件会在运行时被优化，优化后的文件将被保存在缓存中。</p>

2. 关于Zygote虚拟机进程
<p>
&emsp;Zygote是一个虚拟机进程，同时也是一个虚拟机实例的孵化器，每当系统要求执行一个 Android应用程序，Zygote就会FORK出一个子进程来执行该应用程序。这样做的好处显而易见：Zygote进程是在系统启动时产生的，它会完成虚拟机的初始化，库的加载，预置类库的加载和初始化等等操作，而在系统需要一个新的虚拟机实例时。<br/>
&emsp;Zygote通过复制自身，最快速的提供个系统。另外，对于一些只读的系统库，所有虚拟机实例都和Zygote共享一块内存区域，大大节省了内存开销。
</p>

3. dex文件格式

![](http://i2.piimg.com/567571/f7db349234a76ce0.png)

## apk文件 与 dex文件 的关系
1. 通过Android打包工具（appt） ： dex文件 +  资源文件 +  AndroidManifest.xml ==  apk文件

2. Apk打包过程
    1. 打包.class文件为单一DEX文件并运行于Dalvik虚拟机
    2. DEX文件打包进APK文件中（本质上是jar或zip文件）

    ![](http://i2.piimg.com/567571/a13ff962c727a7cb.png)

3. Apk安装过程
    1. 安装时，系统提取DEX文件进行检查和验证。
    2. 第一次运行时，系统完成DEX优化，转换成odex文件。
    3. odex文件存放在/data/dalvik-cache目录并在执行时加载进内存执行。

    ![](http://i2.piimg.com/567571/f34896093441f353.png)

---

##### 参考文章
http://blog.163.com/yglzz_163/blog/static/388555201242762822593/
<br/>
http://blog.chinaunix.net/uid-24439730-id-355883.html
<br/>
http://bbs.pediy.com/showthread.php?t=177114
