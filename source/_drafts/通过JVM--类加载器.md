---
title: 通关JVM--类加载器
date: 2020-03-26 20:26:00
categories:
- Java
- JVM
tags:
- JVM
---

# 5. 类加载器
JDK 8 为例：

名称 | 加载哪的类 | 说明
--- | --- | ---
Bootstrap ClassLoader【再无parent，本身属于C++ level】 | JAVA_HOME/jre/lib | 无法直接访问
Extension ClassLoader | JAVA_HOME/jre/lib/ext 【扩展目录】 | 上级为 Bootstrap，显示为 null
Application ClassLoader | classpath【类路径下】 | 上级为 Extension
自定义类加载器 | 自定义 | 上级为 Application


加载机制为：双亲委派机制，就是“我要问问我爸妈是否已经买了菜，没买，我再去买。”

## 5.1 启动类加载器
这里我们通过一个例子，去修改 启动类加载器的 加载目录， 让它加载本该由 `AppClassLoader`去加载的类：
1. 首先，准备一个`F.java`，
    ```
    public class F {
        static {
            System.out.println("F init.........");
        }
    }
    ```
2. 并将其编译成`.class`文件：
    ```
    javac -g F.java
    ```
3. 然后，让main方法去调用`ClassLoader.loadClass()`
    ```
    public class Load5_1 {
        public static void main(String[] args) throws ClassNotFoundException {
            Class<?> clazz =  Class.forName("com.example.springbootstarterdemo.F");
            System.out.println(clazz.getClassLoader());
        }
    }
    ```
4. 最后，我们尝试修改初始类加载器的加载目录
    ```
    java -Xbootclasspath/a:. com.example.springbootstarterdemo.Load5_1
    ```
    > `-Xbootclasspath` 表示设置 初始类加载器的 加载目录环境变量 `bootclasspath`
    > * 其中 `/a:.`表示将当前目录追加到 `bootclasspath`之后
    > * 可以用这个办法替换核心类：
    >   * `java -Xbootclasspath:<new bootclasspath>`
    >   * `java -Xbootclasspath/a:<追加路径>`
    >   * `java -Xbootclasspath/p:<追加路径>`
5. 最终得到输出结果，输出为`null`，表示是通过来自C++ Level的初始类加载器加载的。
    ```
    F init.........
    null
    ```

关于