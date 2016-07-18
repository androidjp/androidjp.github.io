---
title: ReactiveX 和 RxJava
tags:
- RxJava
categories:
- Rx
- Java
---

## Rx介绍
---
  ReactiveX 简称 Rx。
  Rx = Observables【用于表示异步数据流】 + LINQ【用它的操作符查询异步数据流】 + Schedules【参数化异步数据流的并发处理】

  Rx用到的设计模式精华：观察者模式、迭代器模式

  RxJava中最重要的是：Observable【被观察者，事件源】+ Subscriber【观察者，订阅者】
> 注意：Subscribe\<T\> 是实现 Observable\<T\> 和 Subscription 的一个抽象类，在调用subscribe\(params\)方法时，如果这个params类型为Observer\<T\>，则最终它会转成Subscriber\<T\>，同时，此方法会返回一个Subscription对象，用于调用unsubscribe\(\)方法解绑。


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
  1. just：如果只是调用

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
  * RxJava内置的Scheduler：
    * `Schedulers.immediate()`:默认模式。直接使用当前线程运行。
    * `Schedulers.newThread()`:总是启动新线程，并在新线程中运行。
    * `Sched.io()`:I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。行为模式和 newThread() 差不多，区别在于 io() 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。不要把计算工作放在 io() 中，可以避免创建不必要的线程
    * `Schedulers.computation()`: 计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。
    * `AndroidSchedulers.mainThread()`:它指定的操作将在 Android 主线程运行。
3. Obervable.subscribeOn\(Scheduler\):让call方法以及之前的操作，发生在指定的线程中运行
4. Obervable.observeOn\(Scheduler\):让call之后的回调操作例如map、onNext等操作，发生在指定的线程中运行。

## 数据转换处理
---
> 在事件传递过程中，如果观察者有需要，还可以通过数据转换处理，将传入的数据进行加工或调用，得到更多不同类型的信息。
RxJava提供给我们：map，flatMap来支持数据的‘一对一’和’一对多‘的转换。

1. map

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
  ```
  ///如下过程，将Student[]列表中每个学生的所有课程都一一输出（Student（一）----Course（多））
  Student[] students = ...;
Subscriber<Course> subscriber = new Subscriber<Course>() {
    @Override
    public void onNext(Course course) {
        Log.d(tag, course.getName());
    }
    ...
};
Observable.from(students)
    .flatMap(new Func1<Student, Observable<Course>>() {
        @Override
        public Observable<Course> call(Student student) {
            return Observable.from(student.getCourses());
        }
    })
    .subscribe(subscriber);
  ```
  > 知识点：
  1. flatMap：
    原理：
      * 1)使用传入的事件对象创建一个 Observable 对象
      * 2)并不发送这个 Observable, 而是将它激活，于是它开始发送事件
      * 3)每一个创建出来的 Observable 发送的事件，都被汇入同一个 Observable ，而这个 Observable 负责将这些事件统一交给 Subscriber 的回调方法
    结果：
      * 把事件拆成了两级，通过一组新创建的 Observable 将初始的对象『铺平』之后通过统一路径分发了下去。而这个『铺平』就是 flatMap() 所谓的 flat。
  2. Funx 和 Actionx：
    * 'x'的意义：从0开始，表示有x个参数的Fun()和Action方法。



## 参考文章
---
1. http://gank.io/post/560e15be2dca930e00da1083
2. https://mcxiaoke.gitbooks.io/rxdocs/content/Observables.html
3. http://blog.csdn.net/lzyzsd/article/details/41833541
