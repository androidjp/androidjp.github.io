---
title: Android字典（二）-- 获取系统各个目录
date: 2016-07-22 22:21:00
categories:
- Android
- 系统目录
tags:
- Android系统目录
---

> 引言： 下面列出Android系统在编译打包后由镜像文件生成的Android系统各级主要目录，以及在App中获取各个目录的说明示例。

<!--more-->

## 一、Android系统目录结构
---
* **/init  【系统启动文件】**
* **/system**
  * app【系统应用安装目录】
  * bin【常用的系统本地命令（二进制），大部分是toolbar的链接（类似于嵌入式Linux中的busybox）】
  * etc【系统配置文件，如hosts】
  * font【字体目录】
  * framework【Java平台架构核心库，jar包和odex优化的文件】
  * lib【系统底层共享库，.so库文件】
  * xbin【不常用的系统管理工具，相当于linux 的/sbin】
  * media
    * audio【铃声，提示音 等音频文件， .ogg】
      * notifications【通知】
      * ui【界面】
      * alarms【警告】
      * ringtones【铃声】
  * usr【用户文件夹】
    * keychars
    * keylayout
    * share
    * srec【配置】
    * 等等
  * vendor
  * build.prop【系统设置和变更属性】
* /etc --> /system/etc
* /vendor --> /system/vendor
* /dev【存放设备节点文件】
* /proc【全局系统信息】
* **/data**【用户软件和各种数据】
  * local/tmp【临时目录，无权限要求】
  * app【普遍程序安装目录】
  * system
    * location【其中的location.gps记录最后的坐标，LocationManager.getLastKnownLocation()数据来自此处】
  * data
    * <package_name>
      * files【Context.getFilesDir()， Context.getFileOutput()】
      * cache【Context.getCacheDir() , **系统会在内存不足或者目录大小达到特定数值时自动清理。**】
      * shared_pref【Context.getSharedPreferences()建立的 SharedPreferences文件存放目录】
  * anr【应用在发生ANR 时，Android将问题点的堆栈写入traces.txt文件中】
  * location
    * gps【GPS location provider配置】
  * property【其中persist.sys.timezone记录系统临时区】
* **/sdcard** --> /storage/emulated/legacy   【SD卡的FAT32文件系统挂载到这个目录】
* **Android**
  * **data**
    * <package_name>  【应用的额外数据，应用卸载时自动删除】
      * files【Context.getExternalFilesDir()获取 。 设置 → 应用 → 具体应用详情→ **清除数据** 的操作对象】
      * cache【Context.getExternalCacheDir()获取 。 设置 → 应用 → 具体应用详情→ **清除缓存** 的操作对象】
* lost+found
  *   yaffs文件系统固有的，类似于回收站的文件夹。
* ODEX
  *   从apk中提取出来的可运行文件，即原apk中classes.dex通过dex优化生成的一个单独存放的dex文件。启动应用时不需要再从apk包中提取dex，速度更快。还可以删除apk包中的dex减少体积。缺点是体积变大，而且升级某个给Odex的应用可能会出现问题。

## 二、获取系统各个目录
---
> 以包名为“com.androidjp.app”的应用示例实测得到以下结果，模拟器和真机结果一致。

***Environment.getExternalStorageDirectory().getAbsolutePath():***
结果：/storage/emulated/0
***Environment.getExternalStoragePublicDirectory("").getAbsolutePath(): ***
结果：/storage/emulated/0
***MyAppl.getContext().getPackageName(): ***
结果：com.androidjp.app【你的app的包目录】
***Environment.getDownloadCacheDirectory().getAbsolutePath(): ***
结果：/cache
***Environment.getRootDirectory().getAbsolutePath(): ***
结果：/system
***Environment.getDataDirectory().getAbsolutePath(): ***
结果：/data
***MyAppl.getContext().getFilesDir().getAbsolutePath(): ***
结果：/data/user/0/com.androidjp.app/files
***Environment.getExternalStoragePublicDirectory("files").getAbsolutePath(): ***
结果：/storage/emulated/0/files
***MyAppl.getContext().getExternalFilesDir("").getAbsolutePath(): ***
结果：/storage/emulated/0/Android/data/com.androidjp.app/files
***MyAppl.getContext().getCacheDir().getAbsolutePath(): ***
结果：/data/user/0/com.androidjp.app/cache
***Environment.getExternalStoragePublicDirectory("cache").getAbsolutePath(): ***
结果：/storage/emulated/0/cache
***MyAppl.getContext().getExternalCacheDir().getAbsolutePath(): ***
结果：/storage/emulated/0/Android/data/com.androidjp.app/cache
