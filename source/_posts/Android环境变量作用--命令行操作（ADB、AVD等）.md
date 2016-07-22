---
title: Android环境变量作用--命令行操作（ADB、AVD等）
date: 2016-07-21 09:33:24
categories:
- Android
tags:
- Android命令行
---

> 我们配置Android环境变量时，像SDK中的tools目录和platform-tools目录可以一同配置，这样就可以在终端使用命令行来使用这些Android SDK拥有的工具了。

<!--more-->

## Android环境变量配置
---
> 这些我们应该自己配置过。

  * **Linux**：
    执行：`sudo gedit /etc/profile`
    然后：在/etc/profile 文件中添加如下配置：
    ```
  #set java environment-------------------------------------------------
  JAVA_HOME=/home/androidjp/Android/jdk/jdk1.8.0_25
  export JRE_HOME=/home/androidjp/Android/jdk/jdk1.8.0_25/jre
  export CLASSPATH=.:$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH
  export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
  #set android environment--------------------------------------------
  #set 'tools' and 'platform-tools' are for android avd and adb logcat
  export ANDROID_HOME=/home/androidjp/Android/sdk/android_sdk
  export PATH=/home/androidjp/Android/sdk/android_sdk/tools:/home/androidjp/Android/sdk/android_sdk/platform-tools:$PATH
  #set android aapt enviorment-----------------------------------------
  export AAPT_HOME=$ANDROID_HOME/build-tools/21.1.0/aapt
    ```
  * **Windows**：
  在“我的电脑”->“属性”->“高级”->“环境变量” 中可以配置系统变量*Path*，加上几句，最终得到：
  Path：`……; D:\Android\sdk;D:\Android\sdk\tools;D:\Android\sdk\platform-tools;`


## 命令行管理模拟器设备（AVD）
---
  * android list：列出机器上所有已经安装的Android版本和AVD设备
  * **android list avd**：列出机器上所有已经安装的AVD设备；
  * android list target：列出机器上所有已经安装的Android版本
  * android create avd:创建一个AVD设备
    格式：android create avd -n `<`AVD名称`>` -t `<`SDK版本号`>` -s `<`AVD皮肤`>` -p `<`AVD保存路径`>`
    如：android create avd -n 1.5 -t 3 -s HVGA
  * android delete avd:删除一个AVD设备
  * android update avd:升级一个AVD设备使其符合新的SDK环境
  * android create project：创建一个新的Android项目
  * android update project：更新一个已有的Android项目
  * android create test-project：创建一个新的Android测试项目
  * android update test-project：更新一个已有的Android测试项目

## 命令行启动模拟器
---
  使用emulator.exe启动模拟器的两种方法：
  * emulator -avd `<`AVD名称`>`
  * emulator -data `<`镜像文件名称`>` 【镜像文件一般位于AVD设备保存位置的avd文件夹目录下】
  > avd的默认路径（win）：C:\用户\Admin\.android\avd\avd名.avd

## 常用的ADB命令
---
> ADB是一个非常强大的工具，位于SDK安装目录的platform-tools子目录下，它既可以完成模拟器文件与电脑文件的相互复制，也可以安装apk应用，甚至直接切换到Android系统中执行Linux命令。

  * **adb -devices**：查看当前运行的模拟器
  * adb push c:/123.doc /sdcard/：将电脑文件复制到模拟器中
  * adb push /sdcard/abc.txt c:/：将模拟器文件复制到电脑
  * adb shell：启动模拟器的shell窗口，此时就可以在模拟器的shell窗口中直接执行Linux命令
  * **adb install [-r] [-s] <文件>**：安装apk文件，其中-r表示重装该apk，-s表示将apk安装到SD卡上，默认是安装到内部存储器上
  * **adb uninstall [packge] [-k]**：从系统中卸载程序包，-k表示只删除该应用程序，但保留该应用程序所有的数据和缓存目录
