---
title: Android Material Design实测（一）：TabLayout + RecyclerView + CardView
---
> Material Design 扁平化设计，是AndroidUI设计的主流，下面，利用本人的一个关于MD 基本用法的Demo 点击下载demo.apk（项目地址：[]()） ,来给大家带来一个简单的MD式开发模板。

<!--more-->

### 简单分析三者的优劣势
1. TabLayout
  * 优势：方便我们实现Indicator、配合ViewPager，可以比较方便地实现一个标签界面
  * 劣势：

2. RecyclerView
  * 优势：相比ListView，强制要求列表Item缓存复用机制，标准化了ViewHolder，让ViewHolder充当原来ListView中的convertView；并且，RecyclerView能更好、更加迅速地适应不同的列表布局。
  * 劣势：相比ListView，没有了ListView的头布局设置和ListView的emptyView 设置（当列表为空时，ListView中可设置为空时显示的布局，而RecyclerView则需要另想办法）；整体结构更加复杂，需要考虑①RecyclerView.Adapter ②LayoutManager ③ViewHolder等几个部分

3. CardView
  * 优势：让列表Item更加美观（悬浮阴影效果、圆角边框效果等）
  * 劣势：需要在xml中配置多个自定义app属性，整个组件代码量较大。



##### 参考文章：


http://mp.weixin.qq.com/s?__biz=MjM5NDkxMTgyNw==&mid=2653057706&idx=1&sn=40660d971b1a385bbfcba1a277e0c85b&scene=0#wechat_redirect
