---
title: Android开发细节--查漏补缺（三）：重温Service、BoardcastReceiver和ContentProvider
date: 2016-07-19 10:22:00
categories:
- Android
tags:
- Android
- 面试
- 基础
---
> 引言： Activity在日常开发中用的比较多，那么，除了Activity，后三大组件：Service、BoardcastReceiver、ContentProvider，那就只在一些比较特殊需求的app设计开发中运用到。
  * Android四大组件，除了BroadcastReceiver，其他三种组件都必须在AndroidManifast中注册，对于BroadcastReceiver，既可以在AndroidManifast中注册，也可通过代码形式注册。

  接下来，通过这三大组件各自的运用示例，来重温三个组件。
  




Service（服务）
---
一、运行状态
  计算型组件，一般在后台执行一系列计算任务。Service拥有两种状态，**启动态**和**绑定态**(详见后面的图解)。 启动态时，Service不受启动者Activity的生命周期限制，绑定态则随着启动者Activity的销毁和销毁且过程中容易通信。Service也是一个组件，意味着一般情况下，Service也在主线程中跑，所以，耗时的任务同样需要在Service中建立新的子线程去完成。


BroadcastReceiver（广播）
---
一、运行状态
  消息型组件，用于在不同的组件乃至不同的应用间传递消息。广播的两种
