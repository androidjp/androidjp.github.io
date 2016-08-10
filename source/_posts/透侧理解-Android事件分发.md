---
title: 透侧理解-Android事件分发
date: 2016-07-06 17:04:00
tags:
- Android
categories:
- Android
- 事件分发

---
> 网上看过不少文章，也看过书籍“Android群英传”中有关事件分发的片段，发现，其实总结起来，Android的事件分发机制并不难，下面，给大家分享一个简介易懂的Android事件分发过程。并通过一个简单实际的Demo，来加以说明

<!-- more -->
## 事件分发过程简单图解
---
> 滑动、点击、长按、松开等，归根结底都是触摸事件，而像点击“Back”或者“Home”等，那就归为 onKeyDown事件处理中。


### View的事件分发过程
#### 图解
  ![View的事件分发过程](http://i2.piimg.com/567571/d4392bae47c97773.png)
#### 简单过程
  1. View接收到触摸事件，直接执行onDispatchTouchEvent
    * ①判断View本身是否处于enabled状态（enable为true与否？）：是，到②
    * ②onTouchListener是否为null，否，到③
    * ③执行onTouchListener.onTouch(),如果返回为true，则事件被消费了，整个过程结束
  2. 上面三个小步骤中：①View的enabled为true 或者 ② onTouchListener为null 或者 onTouch()返回为false，那么，都执行onTouchEvent(event)
    * 判断View是否可点击或长按，可以，那么，onTouchEvent一定返回true，事件一定在这里被消耗
    * 查看是否有代理，有，就把事件给代理去处理，onTouchEvent一定返回true，事件一定在这里被消耗
    * 如果不能点击、不能长按，那么，事件无法在这里就被消费，还要最后给回父ViewGroup处理
    * 如果可以点击或者长按，那么，过了clickable和代理（mTouchDelegate）的判断后，开始了不同ACTION的处理：
      * 如果view的enabled为true，并且DOWN事件持续了一定时间，那么，onLongClickListener.onLongClick()执行
      * 如果view的enabled为true，并且onLongClickListener.onLongClick()方法返回值为false，并且UP事件处理时，那么，onClickListener.onClick()执行

  > 注意：
    1. 当View的enabled为false时，所有与触摸事件的相关的Listener的方法都不会执行
    2. 不论View是否可点击，默认情况下【onTouch()方法返回false时】View的onTouchEvent()总会执行
    3. 只要View是可点击或可长按的，onTouchEvent()一定会返回true，表示事件一定会在此被消耗。



### ViewGroup配合子View的事件分发过程
#### 图解
  ![ViewGroup配合子View的事件分发过程](http://i4.piimg.com/567571/d3e0a34d6ebfa732.png)
#### 简单过程
  1. 每个触摸事件产生，都会首先从根布局开始，将事件进行分发，可以它自己处理这个事件（把他拦截，不给子View），也可以分发给它的子View/ViewGroup去处理
  2. 执行ViewGroup的dispatchTouchEvent，对于每一个ACTION，都会有一个IF块，每个事件都会做两个判断：①是否允许自身去拦截 ②是否拦截
  3. 在上述的“②是否拦截”中，就是执行ViewGroup的onInterceptTouchEvent 方法，看其返回值是否为true，是，则进行事件拦截
  4. 如果不拦截该事件，那么：
    * DOWN事件：获取当前x、y坐标的子View，赋值给mMotionTarget , 然后执行mMotionTarget.dispatchTouchEvent (也就是找到可以处理该事件的子View
      ，然后递归地让子View做事件分发)
    * MOVE事件：如果mMotionTarget不为null，直接执行mMotionTarget.dispatchTouchEvent 让他去分发这个MOVE事件
    * UP事件：在给子View分发事件前，修改坐标系统，把当前触摸位置的x、y分别减去child.left 和 child.top,再传给child【此过程将点击的位置的相对对象，从当前ViewGroup换成子View】
  5. 如果拦截该事件，则自己执行onTouchEvent进行事件处理

  > 注意：实际的View上的点击事件，如果整个经过过程，是从父ViewGroup的事件分发，到子View上的事件分发处理，再返回到父ViewGroup的过程【V形】

## 事件的拦截
---
### ViewGroup 如何实现拦截
ViewGroup.onInterceptTouchEvent(MotionEvent ev)
```
@Override  
    public boolean onInterceptTouchEvent(MotionEvent ev)  
    {  
        int action = ev.getAction();  
        switch (action)  
        {  
        case MotionEvent.ACTION_DOWN:  
            //如果你觉得需要拦截  
            return true ;   
        case MotionEvent.ACTION_MOVE:  
            //如果你觉得需要拦截  
            return true ;   
        case MotionEvent.ACTION_UP:  
            //如果你觉得需要拦截  
            return true ;   
        }  

        return false;  
    }  
```
由于此方法返回true时，倒置mMotionTarget为null，无法往子View传递事件。
优先级：DOWN -> MOVE -> UP
* 在DOWN return true:子View的 DOWN,MOVE,UP过程都不会捕获事件
* 在MOVE return true：子View的MOVE，UP过程都不会捕获事件

  子View如何让父ViewGroup不去拦截某个事件：

  通过：
```
requestDisallowInterceptTouchEvent(boolean)
```
设置是否允许父ViewGroup拦截，具体写法如下。

```
@Override  
    public boolean dispatchTouchEvent(MotionEvent event)  
    {  
        getParent().requestDisallowInterceptTouchEvent(true);    
        int action = event.getAction();  

        switch (action)  
        {  
        case MotionEvent.ACTION_DOWN:  
            Log.e(TAG, "dispatchTouchEvent ACTION_DOWN");  
            break;  
        case MotionEvent.ACTION_MOVE:  
            Log.e(TAG, "dispatchTouchEvent ACTION_MOVE");  
            break;  
        case MotionEvent.ACTION_UP:  
            Log.e(TAG, "dispatchTouchEvent ACTION_UP");  
            break;  

        default:  
            break;  
        }  
        return super.dispatchTouchEvent(event);  
    }  
```
### 原理
这是ViewGroup MOVE 和 UP 拦截的部分源码：
```
if (!disallowIntercept && onInterceptTouchEvent(ev)) {  
            ………………

            return true;  
        }  
```
从上面可以看到，当子View中设置了不允许父ViewGroup去拦截，那么，相当于设置了这里的disallowIntercept 为 true，于是，上面源码的if块直接被忽略了，从而实现了不拦截。

> 注意： 如果在父ViewGroup的 onInterceptTouchEvent(event) 方法中，关于ACTION_DOWN 的事件判断，直接返回true，那么，子View是无法获取到ACTION_DOWN 的事件的。


### 特殊情况
当ViewGroup找不到分发事件的子View：
* 情况：当ViewGroup的dispatchTouchEvent()中的ACTION_DOWN事件的方法过程如下：
  ```
  if (child.dispatchTouchEvent(ev))  {  
                                // Event handled, we have a target now.  
                                mMotionTarget = child;  
                                return true;  
                            }  
  ```
  这里，如果子View的dispatchTouchEvent(ev)为false，也就是没有子View去接收这个DOWN事件或者说子View不可点击和长按，于是，这个DOWN事件就没有处理掉，那么，ViewGroup本身的dispatchTouchEvent对于这个DOWN事件，也返回false，表示需要再给到ViewGroup的父类去做，那么，我们知道，ViewGroup父类是View，于是，最终，这个DOWN事件，还是给到了ViewGroup本身来做。
  （我们没有找到做这个事件的target，就意味着我们自己处理了～～）
  > 当子View可点击或长按，那么，就必定能够自己消费这些事件。


## Demo下载
---
  本人上传了一个简单的测试Demo，用于测试事件分发过程，还有疑惑的朋友可以下载Demo来跑跑，看看整个log过程，会更加理解整个Android事件分发过程。
  Demo github地址：https://github.com/androidjp/jp-event-demo
  >  Demo项目中gradle版本为2.2.1，插件版本为1.5.0，另外，demo导入了库：com.jakewharton:butterknife:7.0.1

  Demo主界面如下：
  ![](http://i1.piimg.com/567571/453e3e2a3c8da57d.png)



## 小结
---
* 触摸事件的分发处理过程：父ViewGroup去分发事件(分发当中想拦截就拦截) --> 子View接收到事件 --> 子View看看自己是否可以做这个事（是否可点击/长按等） --> 子View去尝试做了这个事（有监听，看onTouchListener.onTouch返回值） -->最后，如果子View可以搞定（listener.onTouch或者onTouchEvent返回true），事件就被消费 ，否则，将事件抛回给父ViewGroup
* onLongClickListener.onLongClick() 和 onClickListener.onClick() 是在onTouchEvent中进行判断和调用的
* 如果ViewGroup找到了能够处理该事件的子View，则直接给子View处理，自己的onTouchEvent不会被触发。
* 如果ViewGroup重写自己的onInterceptTouchEvent(event),并去拦截了一些事件（onInterceptTouchEvent返回true），那么，ViewGroup则自己去处理这些事件，执行自己的onTouchEvent方法
* 子View 通过调用getParent().requestDisallowInterceptTouchEvent(true); 阻止ViewGroup对其MOVE或者UP事件进行拦截
* 子View不管可不可以点击（是否clickable），只要手势触摸到它的范围，就会激发其onTouchEvent事件
* 当子View的enabled为false时，无论如何都不会执行onTouchListener.onTouch()方法、onClickListener.onClick()方法、onLongClickListener.onLongClick()方法等

## 细节与注意点
---
1. 消费了Down事件的view称为‘目标View’，目标View可接收后续事件（此时，非目标view只要不去拦截它的后续事件，那非目标view不会接收后续事件）
2. view或者viewGroup想要处理后续事件，就必须消费DOWN事件。当DOWN事件没有从父传到子的过程中没有被消耗过，那么他的后续MOVE事件和UP事件都将经过一个轮“V”字形传递过程后，回到Activity处，这时由Activity自己来接收和处理【也就是说，Activity自己觉得，我找不到一开始消耗DOWN事件的目标view，那我的后续事件肯定也找不到，所以我也不需要再往下分发MOVE和UP事件了，直接我自己来吧，处理得了就处理，处理不了就不管啦，dispatch到天上去】
3. DOWN事件被子view消费了，但后续的MOVE事件被view的父亲viewgroup拦截，此时，ViewGroup将代替子view称为新一代目标view。【因为，有DOWN事件被消费过， 那么Activity就知道有个TargetView存在，那么，在外界有后续事件传进来时，Activity会将事件分发到下面来，让这个TargetView（Activity不知道它是谁，只知道它存在）去获取并消费它，此时，ViewGroup去拦截这些后续事件，那么，首先：①targetView是子View，是事件要去找的目标，但是，找不到（因为它被拦截了），②于是，它选择暂时让拦截了它的父ViewGroup充当起TargetView】
4. UP事件与MOVE事件等都属于DOWN的后续事件，情况与总结三类似。
5. 如果总结三的“拦截”不存在，那么，子View就会一直被当做TargetView，不管它处不处理后续事件，这些后续事件都不会给别人处理了（“V”过程，就算返回给上层ViewGroup（不包括Activity），这时，也是路过而已，上层不会去处理它的）
6. 从<u>第二点</u>和<u>第五点</u>中得出的，只要子view没有消费touch事件，在分发调用返回时，最终会给回Activity本身，让Activity自己来处理，不管Activity有没有能力消费此事件。
7. 子view调用 `getParent().requestDisallowInterceptTouchEvent(true)`方法请求父类禁止拦截事件，那么这个方法会递归地请求所有的父类都禁止拦截事件（也就是让上层都收手）。而如果在Down事件的时候只是请求父View禁止拦截但子view本身又不消费Down事件（相当于，我叫爸爸别去管这件事，而我自己又不想管这件事，最终，这件事和后续的事都不会再给爸爸和我管了，Activity一怒之下自己管了），虽然父View不再拦截了，但后续事件也接收不到了(这个拦不拦截没有多大的意义，因为没有消费down事件，所以并不是目标view了，事件也就传递不到它了)。
8. 如果两个View不是包含关系，而是重叠关系，比如：viewA压在viewB上面， 那么，上层的ViewA先拿到事件，如果消费了，那么事件不会传给viewB，否则，事件返回给调用处，接着再传递给viewB。
9.
  * 父类的onTouchEvent方法要想执行，要么是等所有的子View都不消费Down事件，要么是父View把事件拦截。
  * 如果子类消费了Down事件，而父View又想处理这个事件，则父View可以在dispatchTouchEvent方法处理touch事件
  * 如果子View请求了禁止父View拦截，且父View还想要拦截的话，可在父View的dispatchTouchEvent方法中不调用super.dispatchTouchEvent则把事件拦截了

## 用法举例
---
1. 比如你需要写一个类似slidingmenu的左侧隐藏menu，主Activity上有个Button、ListView或者任何可以响应点击的View，你在当前View上死命的滑动，菜单栏也出不来；因为MOVE事件被子View处理了~ 你需要这么做：在ViewGroup的dispatchTouchEvent中判断用户是不是想显示菜单，如果是，则在onInterceptTouchEvent(ev)拦截子View的事件；自己进行处理，这样自己的onTouchEvent就可以顺利展现出菜单栏了
2. 如果你需要用到 （左右滑动切换Tab界面+上下滑动界面内列表） 时，这里很可能就出现滑动卡卡顿顿或者滑动失灵的问题，就需要根据你手势滑动的总体角度是横向还是纵向，来判断是切换tab页面还是滚动列表。



## 参考文章
---
[可能是讲解Android事件分发最好的文章](http://www.jianshu.com/p/2be492c1df96)
[鸿洋的ViewGroup事件分发](http://blog.csdn.net/lmj623565791/article/details/39102591)
