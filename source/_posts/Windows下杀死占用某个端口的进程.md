---
title: Windows下杀死占用某个端口的进程
date: 2016-07-20 14:35:24
categories:
- Windows
- bug解决
tags:
- Windows
- 进程杀死
---
> 很多情况下，例如：开发Java有关Socket等网络通信的程序时，我们利用某个Port进行调试连接是否成功的过程中，会出现java.net.BindException: Address already in use: JVM_Bind 的错误。
  下面，简单说明整个解放某个端口的过程（[原文章](http://jingyan.baidu.com/article/4dc40848b4fac8c8d946f1eb.html)）：

<!--more-->

## 一、简单除暴
---
  1. 方法一：重启电脑（麻烦）（实际上关联着：重置所有相关的虚拟器）
  2. 方法二：重启编译器（有时会失效）

## 二、命令行方法
---
  1. 首先，调出终端：
    * 方法一：Windows键+r,输入cmd
    * 方法二：开始-->搜素“cmd”,点击运行
    * 方法三：开始-->运行，输入cmd
  2. 终端输入：***netstat -ano***
        目的：输出所有被占用的端口
        每一列分别表示：*协议 | 本地地址 | 外部地址 | 状态 | PID*
  3. 终端输入：***netstat -ano | findstr 1234***
        目的：输出所有的1234端口，从而**查看最后一列它的PID是多少**。
        注意：有的电脑会提示：“'netstat '不是内部或外部命令，也不是可运行的程序或者批处理文件”
        解决方法：那是因为操作不在系统system32文件夹下，所以只需要输入：
        ***cd c:\windows\system32\***
        回车，然后再接着输入即可
  4. 终端输入：***taskkill  /f /pid  4567***【1234端口对应的PID】
  5. 成功杀死进程。
