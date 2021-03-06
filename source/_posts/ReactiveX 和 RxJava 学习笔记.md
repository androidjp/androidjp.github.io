---
title: ReactiveX 和 RxJava 学习笔记
date: 2016-07-12 17:43:00
tags:
- RxJava
categories:
- Rx
- Java
---

> 引言：
学习了一下RxJava，理解其是一个以升级版的观察者模式为核心的异步处理库。旨在以更加简介、可读性更强的代码去实现数据异步处理和线程前通信。
下面，是本人对RxJava基础的学习笔记和总结，算是入门级别。

<!--more-->

## Rx介绍
---
  ReactiveX 简称 Rx。
  Rx = Observables【用于表示异步数据流】 + LINQ【用它的操作符查询异步数据流】 + Schedules【参数化异步数据流的并发处理】
  Rx用到的设计模式精华：观察者模式、迭代器模式
  RxJava中最重要的是：Observable【被观察者，事件源】+ Subscriber【观察者，订阅者】

## RxJava图解
---
  可先通过图解总览大概：
  RxJava之观察者模式的基本运作过程，如下：
  ![RxJava之观察者模式的基本运作过程](http://i2.piimg.com/567571/e01494ea59380277.png)
  RxJava观察者模式顺序图，如下：
  ![RxJava观察者模式顺序图](http://i4.piimg.com/567571/1c2b91be67aab8a5.png)
> 注意：`Subscribe<T>` 是实现 `Observable<T>` 和 `Subscription` 的一个抽象类，在调用`subscribe(params)`方法时，如果这个`params`类型为`Observer<T>`，则最终它会转成`Subscriber<T>`，同时，此方法会返回一个`Subscription`对象，用于调用`unsubscribe()`方法解绑。


## 单线程中RxJava基本用法和例子
---
### 1. RxJava的几种基本写法（观察者模式）
#### 方式一：
  原始的观察者模式写法，如下：
```
///被观察者
Observable<String> myObservable = Observable.create(
                new Observable.OnSubscribe<String>(){

                    @Override
                    public void call(Subscriber<? super String> subscriber) {
                        subscriber.onNext("hello world");
                        subscriber.onCompleted();
                    }
                }
        );

///观察者
Subscriber<String> mySubscriber = new Subscriber<String>() {
          @Override
          public void onCompleted() {}

          @Override
          public void onError(Throwable e) {}

          @Override
          public void onNext(String s) {
                Toast.makeText(MainActivity.this, s, Toast.LENGTH_SHORT).show();
            }
      };

///订阅（让两者产生关联,并启动）
 myObservable.subscribe(mySubscriber);
```
#### 方式二：
  相对方式一，化简定义方法体的部分，使用Action来实现不完整回调，结果如下：
```
//被观察者
//等价于： call(String) -> onNext(String)过程只调用一次 ->onCompleted()/onError()
Observable<String> myObservable = Observable.just("Hello world");

///观察者
///调用subscribe()时自动生成Subscriber并调用onNext()
Action1<String> onNextAction = new Action1<String>() {
      @Override
      public void call(String s) {
          Toast.makeText(MainActivity.this, s, Toast.LENGTH_SHORT).show();
      }
};

///观察者
///调用subscribe()时自动生成Subscriber并调用onError()
Action1<Throwable> onErrorAction = new Action1<Throwable>() {
    // onError()
    @Override
    public void call(Throwable throwable) {
        // Error handling
    }
};

///观察者
///调用subscribe()时自动生成Subscriber并调用onCompleted()
Action0 onCompletedAction = new Action0() {
    // onCompleted()
    @Override
    public void call() {
        Log.d(tag, "completed");
    }
};

//////订阅（让两者产生关联,并启动）
 myObservable.subscribe(onNextAction);
 // myObservable.subscribe(onErrorAction);
 // myObservable.subscribe(onCompletedAction);

```
#### 方式三：
  相对方式二，进行链式调用，如下：
```
///省略Obervable对象的创建
Observable.just("this is your sign：")
                ///省略Action1对象的创建，直接匿名内部类方式添加订阅
                .subscribe(new Action1<String>() {
                    @Override
                    public void call(String s) {
                        Toast.makeText(MainActivity.this, s, Toast.LENGTH_SHORT).show();
                    }
                });

```
> 注意：方式三中加了map这个RxJava的映射方法，用于将事件处理的复杂过程给被观察者来做，尽可能地减少观察者的工作。
知识点：
  1. just：如果只是调用:  onNext() 【一到多次】 --> onCompleted()这个过程，那么，可以使用just()快速创建Observable

### 2. 基本应用
#### 1. 打印字符串数组
```
String[] names = ...;
Observable.from(names)
    .subscribe(new Action1<String>() {
        @Override
        public void call(String name) {
            Log.d(tag, name);
        }
    });
```
> Observable.from(params) : params是数组类型的参数，在执行时，会调用Subscriber的onNext方法多次，每次处理一个item，之后，调用onCompleted\(\)或者onError\(\).



#### 2. 通过id获取图片并显示
```
int drawableRes = ...;
ImageView imageView = ...;
Observable.create(new OnSubscribe<Drawable>() {
    @Override
    public void call(Subscriber<? super Drawable> subscriber) {
        Drawable drawable = getTheme().getDrawable(drawableRes));
        subscriber.onNext(drawable);
        subscriber.onCompleted();
    }
}).subscribe(new Observer<Drawable>() {
    @Override
    public void onNext(Drawable drawable) {
        imageView.setImageDrawable(drawable);
    }

    @Override
    public void onCompleted() {
    }

    @Override
    public void onError(Throwable e) {
        Toast.makeText(activity, "Error!", Toast.LENGTH_SHORT).show();
    }
});
```

## 多线程中RxJava的使用
---
> 在 RxJava 的默认规则中，事件的发出和消费都是在同一个线程的。也就是说，如果只用上面的方法，实现出来的只是一个同步的观察者模式。观察者模式本身的目的就是『后台处理，前台回调』的异步机制，因此异步对于 RxJava 是至关重要的。

### 1. 基本写法
```


Observable.just(1,2,3,4)
                ///指定 subscribe() 发生在 IO 线程
                .subscribeOn(Schedulers.io())
                // 指定 Subscriber 的回调发生在主线程
                .observeOn(AndroidSchedulers.mainThread())
                .map(new Func1<Integer, String>() {
                    @Override
                    public String call(Integer integer) {
                        Log.e("TestActivity", "当前线程："+ Thread.currentThread());
                        String res = "字符串："+integer;
                        return res;
                    }
                })
                .subscribe(new Subscriber<String>() {
                    @Override
                    public void onCompleted() {
                        Toast.makeText(TestActivity.this,"完成",Toast.LENGTH_SHORT).show();
                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(String s) {
                        Log.e("TestActivity", "当前线程："+ Thread.currentThread());
                        Toast.makeText(TestActivity.this,s,Toast.LENGTH_SHORT).show();
                    }
                });
```
> 知识点：
  1. just\((1,2,3,4\):
    前者等价于如下代码：
    ```
    Observable.create(new Observable.OnSubscribe<Integer>() {
            @Override
            public void call(Subscriber<? super Integer> subscriber) {
                Log.e("TestActivity", "call当前线程："+ Thread.currentThread());
                subscriber.onNext(1);
                subscriber.onNext(2);
                subscriber.onNext(3);
                subscriber.onNext(4);
                subscriber.onCompleted();
            }
        })
    ```
  2. Scheduler：
    * 背景：在不指定线程的情况下， RxJava 遵循的是线程不变的原则，即：在哪个线程调用 `subscribe()`，就在哪个线程生产事件；在哪个线程生产事件，就在哪个线程消费事件。
    * 概念：调度器（线程控制器）
    * 作用：切换线程传递事件，达到异步的目的
    * RxJava内置的Scheduler：（文章下面会详细总结）
      * `Schedulers.immediate()`:默认模式。直接使用当前线程运行。
      * `Schedulers.newThread()`:总是启动新线程，并在新线程中运行。
      * `Sched.io()`:I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。
      * `Schedulers.computation()`: 计算所使用的 Scheduler。
      * `AndroidSchedulers.mainThread()`:它指定的操作将在 Android 主线程运行。
  3. Obervable.subscribeOn\(Scheduler\):让call方法以及之前的操作，发生在指定的线程中运行
  4. Obervable.observeOn\(Scheduler\):让call之后的回调操作例如map、onNext等操作，发生在指定的线程中运行。

## RxJava常用操作--数据转换处理
---
> 在事件传递过程中，如果观察者有需要，还可以通过数据转换处理，将传入的数据进行加工或调用，得到更多不同类型的信息。
RxJava提供给我们：map，flatMap来支持数据的‘一对一’和’一对多‘的转换。

1. map
  作用：实现数据的一对一转化过程
  以下例子可以说明：
  ```
///省略Obervable对象的创建
Observable.just("this is your sign：")
                ///将传入的参数由String变成String[]
                .map(new Func1<String, String[]>() {
                    @Override
                    public String[] call(String s) {
                        String[] strings = s.split(" ");
                        return strings;
                    }
                })
                ///将传入的参数由String[]变成Integer
                .map(new Func1<String[], Integer>() {
                    @Override
                    public Integer call(String[] strings) {
                        int len = strings.length;
                        return len;
                    }
                })
                ///将传入的参数由Integer变成String
                .map(new Func1<Integer, String>() {
                    @Override
                    public String call(Integer integer) {
                        return integer+"";
                    }
                })
                ///省略Action1对象的创建，直接匿名内部类方式添加订阅
                .subscribe(new Action1<String>() {
                    @Override
                    public void call(String s) {
                        Toast.makeText(MainActivity.this, s, Toast.LENGTH_SHORT).show();
                    }
                });

  ```
2. flatMap
  作用：实现数据的一对多转换过程
  先看如下具体例子：
  ```
  private void testFlatMap() throws CloneNotSupportedException {
        List<Student> studentList = new ArrayList<>();
        ///测试：构建两个Student对象
        Student xiaoming = new Student();
        Student honghong = new Student();
        ///测试：构建Course对象集
        Course chinese = new Course("语文");
        Course english = new Course("英语");
        Course math  = new  Course("数学");

        ///进行赋值操作，这样一来：
        /// xiaoming：id为“2222”，并有两门课程：语文和英语
        /// honghong：id为“007” ，并有两门课程：英语和数学
        xiaoming.id= "2222";
        honghong.id= "007";
        xiaoming.courseList = new ArrayList<>();
        xiaoming.courseList.add(chinese.clone());
        xiaoming.courseList.add(english.clone());
        honghong.courseList = new ArrayList<>();
        honghong.courseList.add(english.clone());
        honghong.courseList.add(math.clone());

        studentList.add(xiaoming);
        studentList.add(honghong);

        ///下面的过程，就是提取：列表中的列表
        Observable.from(studentList)
                .flatMap(new Func1<Student, Observable<Course>>() {
                    @Override
                    public Observable<Course> call(Student student) {
                        Log.e("学生信息", student.id);
                        return Observable.from(student.courseList);
                    }
                })
                .map(new Func1<Course, String>() {
                    @Override
                    public String call(Course course) {
                        return course.name;
                    }
                })
                .subscribe(new Action1<String>() {
                    @Override
                    public void call(String s) {
                        Log.e("course信息",s);
                    }
                });
    }
  ```
  最终得到结果为：
  ![](http://i1.piimg.com/567571/6f2f3e1300243da9.png)


> 知识点：
1. flatMap：
  * 作用：实现传递数据的一对多变换（比如：我想要对一个列表中每一个item都进行一个数据类型转换并输出的操作）
  * 原理：
    * 1)使用传入的事件对象创建一个 Observable 对象
    * 2)并不发送这个 Observable, 而是将它激活，于是它开始发送事件
    * 3)每一个创建出来的 Observable 发送的事件，都被汇入同一个 Observable ，而这个 Observable 负责将这些事件统一交给 Subscriber 的回调方法
    * 结果：把事件拆成了两级，通过一组新创建的 Observable 将初始的对象『铺平』之后通过统一路径分发了下去。而这个『铺平』就是 flatMap() 所谓的 flat。
2. Funx 和 Actionx：
  * 'x'的意义：从0开始，表示有x个参数的Fun()和Action()方法。


## RxJava各方法汇总
---
### 1. 用于创建Observable的操作符：
  * filter() —输出和输入相同的元素，并且会过滤掉那些不满足检查条件的。
  * take() —输出最多指定数量的结果。
  * Delay() —让发射数据的时机延后一段时间
  * Create — 通过调用观察者的方法从头创建一个Observable
  * Defer — 在观察者订阅之前不创建这个Observable，为每一个观察者创建一个新的Observable
  * Empty/Never/Throw — 创建行为受限的特殊Observable
  * From — 将其它的对象或数据结构转换为Observable
  * Interval — 创建一个定时发射整数序列的Observable
  * Just — 将对象或者对象集合转换为一个会发射这些对象的Observable
  * Range — 创建发射指定范围的整数序列的Observable
  * Repeat — 创建重复发射特定的数据或数据序列的Observable
  * Start — 创建发射一个函数的返回值的Observable
  * Timer — 创建在一个指定的延迟之后发射单个数据的Observable

### 2. 用于对Observable发射的数据进行变换的操作符：
  * Buffer — 缓存，可以简单的理解为缓存，它定期从Observable收集数据到一个集合，然后把这些数据集合打包发射，而不是一次发射一个
  * FlatMap — 扁平映射，将Observable发射的数据变换为Observables集合，然后将这些Observable发射的数据平坦化的放进一个单独的Observable，可以认为是一个将嵌套的数据结构展开的过程。
  * GroupBy — 分组，将原来的Observable分拆为Observable集合，将原始Observable发射的数据按Key分组，每一个Observable发射一组不同的数据
  * Map — 映射，通过对序列的每一项都应用一个函数变换Observable发射的数据，实质是对序列中的每一项执行一个函数，函数的参数就是这个数据项
  * Scan — 扫描，对Observable发射的每一项数据应用一个函数，然后按顺序依次发射这些值
  * Window — 窗口，定期将来自Observable的数据分拆成一些Observable窗口，然后发射这些窗口，而不是每次发射一项。类似于Buffer，但Buffer发射的是数据，Window发射的是Observable，每一个Observable发射原始Observable的数据的一个子集

### 3. 线程切换和控制相关操作符：
  * subscribeOn(Scheduler) — 指定事件的call方法以及以前的操作到一个线程中
  * observeOn(Scheduler) — 指定事件的call方法之后的操作（如：map(),onNext(),onCompleted(),onError()）到一个线程中【注意：不包括Subscriber.onStart()方法，该方法在默认它所在的线程中执行】
  * 参数Scheduler有：
    * Schedulers.immediate():默认模式。直接使用当前线程运行。
    * Schedulers.newThread():总是启动新线程，并在新线程中运行。
    * Sched.io():I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。行为模式和 newThread() 差不多，区别在于 io() 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。不要把计算工作放在 io() 中，可以避免创建不必要的线程
    * Schedulers.computation(): 计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。
    * AndroidSchedulers.mainThread():它指定的操作将在 Android 主线程运行。

## 参考文章
---
* http://blog.csdn.net/lzyzsd/article/details/41833541
* https://mcxiaoke.gitbooks.io/rxdocs/content/Observables.html
* http://gank.io/post/560e15be2dca930e00da1083
