---
title: MVP架构的理解
date: 2016-08-11 15:08:00
categories:
- Android
- 架构
tags:
- Android
- MVP架构
---

<!--more-->

首先，放两个网上关于MVP架构的图解，对比一下：
> 图解一：
![](http://upload-images.jianshu.io/upload_images/2369895-b24fbbb7c5735801.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
图解二：
![](http://upload-images.jianshu.io/upload_images/2369895-7919a1f8e616a804.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里，本人会更加倾向与图解一。

## 一、M、V、P的分析
&emsp;M层，也就是Model层（也可以理解成：数据源层、持久层、Data层），是专门给外界提供数据的一层，这里
的‘外界’一般是指本应用的其他层（逻辑处理层和展示层）。
&emsp;V层，指的是View层（界面展示层、与用户交互的界面），是专门用于展示数据、与用户交互的层。
&emsp;P层，指的是Presenter层（逻辑处理层、主持层），是不同与MVC模式的一层，因为它分出来是为了将界面中所有较为复杂的逻辑操作工程分出来，减轻V层的负担，同时，让原本V层与M层的直接关系断开，达到解耦的效果。
&emsp;从M、V、P三者的分析，结合上面的图解可以看到，其实图解一比图解二更好的地方在于：Presenter与Model之间的交互，是可以双向的。而这一点，由于很多开源库或者框架已经实现了关于数据的一个异步获取或处理的过程，所以，在设计MVP架构的时候，很多MVP架构辅助库，并没有着重P和M之间通信的问题，而着重与P和V之间的通信问题，这本来没有错，因为MVP架构出来的初衷本来就是为了减轻View层的逻辑负担。然而这里，我们在遇到一些网络操作、异步处理过程时，可能需要对Presenter和Model间的监听和交互想一想怎么设计了。
## 二、MVP架构优劣分析
* 优点：很容易看出来，就是：
  * 逻辑和展示的分离，Presenter减轻View的负担。【相当于：主持人负责主持好每一个表演的出场和开场白，表演者听命令直接做动作就好】
  * View层与Model层解耦，模块化，易于后期维护。
* 缺点
  * View负担小了，Presenter负担就大的。目前的MVP架构辅助库，大多考虑的是一个Activity/Fragment配一个Presenter，那么，意味着，一个界面，需要做的逻辑操作，全部交给一个Presenter去做，这样一来，Presenter的代码量就很大了。【这里，只能尽量保证每一个界面的功能相对单一，以减少Presenter代码量，但是功能一旦减少，代码本来就少了，还需要Presenter吗？这就存在一个矛盾了】
  * Presenter 与 Model层的交互，需要同样的一个异步处理过程，而这个过程，如果不借助第三方库，自己动手，则同样需要一个回调接口的设计。
