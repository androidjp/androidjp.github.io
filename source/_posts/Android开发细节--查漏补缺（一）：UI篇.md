---
title: Android开发细节--查漏补缺（一）：UI篇
date: 2016-07-18 19:55:00
categories:
- Android
tags:
- Android
- 面试
- 基础
---


> 引言：一开始，先和大家可能从最开始接触Android开发时就存在的一些我们用过但不太关注的UI部件相关知识。第一篇，先说一下有关：Menu、标题栏 和 Android主题（Theme）的相关知识。



## Android Theme（主题）
---
> 注意：一般我们使用或者自定义style等，大多用于单个View，称为View的样式。而样式也能用于Activity或整个Application，这时就需要在相应<activity>或<application>标签中设置android:theme*属性，引用的其实也是style*，但一般称为主题。


* themes.xml: 低版本的主题，目标API level一般在10以下
* themes_hole.xml: 从 API level 11添加的主题

    ![](https://app.yinxiang.com/shard/s49/res/fe5bdb86-3dc5-48b7-b3a1-4cdebb49c0b3.png)

**一、 support v7库**
   作用：用于某些控件和主题等兼容至Android2.1版本（API 7）
   对于Theme主题的影响：Theme.AppCompat系列兼容主题，在引用到v7包之后，就要求使用，而android3.0以上系统带有的Theme.Holo系列主题，则是未引用兼容包时注意使用。
  * Android3.0及以上的版本的Style一般写法：
  ```
  <style name="XXX" parent="@android:style/Theme.Holo">
     <item name="android:xxx">yyy</item>
  </style>
  ```
  * Android2.1及以上兼容版本的Style一般写法：
  ```
  <style name="XXX" parent="@style/Theme.AppCompat">
     <item name="xxx">yyy</item>
  </style>
  ```

**二、 theme测试示例**
  1. 无ActionBar
    * 白色背景： `Activity`+ `android:theme="@style/Theme.AppCompat.Light"`
    * 黑色背景： `Activity`+ `android:theme="@style/Theme.AppCompat"`
    * 自定义默认背景：`Activity`  或者 `Activity`+`android:theme="@style/AppTheme"`

  2. 有ActionBar
    1. 旧版本：继承ActionBarActivity
      * 全白色背景：`ActionBarActivity`+`android:theme="@style/Theme.AppCompat.Light"`
      * 全黑色背景：`ActionBarActivity`+`android:theme="@style/Theme.AppCompat"`
      * 白色背景黑色ActionBar：`ActionBarActivity`+`android:theme="@style/Theme.AppCompat.Light.DarkActionBar"`
      * 自定义默认背景：`ActionBarActivity`  或者`ActionBarActivity`+`android:theme="@style/AppTheme"`


## Menu

---
>  Android Menu 主要是两种：①OptionMenu【点击ActionBar的菜单选项或者手机硬件MENU按键触发显示菜单】 ②ContextMenu【指定的View或ViewGroup的长按触发】

**一、 OptionMenu**
  1. 方法执行流程
    ![OptionsMenu.png](http://upload-images.jianshu.io/upload_images/2369895-06df515625ff68b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  2. 具体实现代码示例
    * Activity代码中添加：

    ```
      @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        //方式一（java）
         /*第一个参数是groupId，如果不需要可以设置为Menu.NONE。
将若干个menu item都设置在同一个Group中，可以使用setGroupVisible()，setGroupEnabled()，setGroupCheckable()这样的方法，
而不需要对每个item都进行setVisible(), setEnable(), setCheckable()这样的处理，这样对我们进行统一的管理比较方便
       * 第二个参数就是item的ID，我们可以通过menu.findItem(id)来获取具体的item
       * 第三个参数是item的顺序，一般可采用Menu.NONE，具体看本文最后MenuInflater的部分
       * 第四个参数是显示的内容，可以是String，或者是引用Strings.xml的ID
       */
        menu.add(Menu.NONE,ONE_ID,Menu.NONE,"1 Pixel");
        menu.add(Menu.NONE, TWO_ID, Menu.NONE, "2 Pixels");
        menu.add(Menu.NONE, EIGHT_ID, Menu.NONE, "8 Pixels");
        ……
         ///方式二（xml）
        MenuInflater mi = getMenuInflater();
        mi.inflate(R.menu.activity_actionbar,menu);

        // 子菜单设置
        //通过addSubMenu设置子菜单，作为item加入Menu。参数和addMenu一致，为了简单，我们这里的ID直接采用数字表示
        SubMenu submenu = menu.addSubMenu(Menu.NONE, 100, Menu.NONE, "子菜单测试");
        //在SubMenu中增加子菜单的item
        submenu.add(Menu.NONE,101,Menu.NONE,"sub One");
        submenu.add(Menu.NONE,102,Menu.NONE,"sub Two");
        submenu.add(Menu.NONE,103,Menu.NONE,"sub Three");
        submenu.add(Menu.NONE,104,Menu.NONE,"sub Four");
        return super.onCreateOptionsMenu(menu);
    }

    @Override
    public boolean onPrepareOptionsMenu(Menu menu) {
        ///----------实现每次Menu打开时的操作---------------
        return super.onPrepareOptionsMenu(menu);
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()){
            case R.id.action_search:
                Toast.makeText(this, "搜索",Toast.LENGTH_SHORT).show();
               break;
            case R.id.action_settings:
                Toast.makeText(this, "设置",Toast.LENGTH_SHORT).show();
                break;
        }
        return super.onOptionsItemSelected(item);
    }
    ```
    * 自定义的res/menu/activity_actionbar.xml：

    ```
    <?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:yourapp="http://schemas.android.com/apk/res-auto" >
    <!-- 搜索, 应该展示为动作按钮 -->
    <!--不使用v7兼容库，那么，就可以使用android:showAsAction属性，否则，无法使用,但是能用自己指定的yourapp:showAction-->
    <!--下面的属性：‘ifRoom’表示如果ActionBar有空间，就显示这个item项；‘never’表示默认不显示-->
    <item android:id="@+id/action_search"
        android:icon="@android:drawable/ic_menu_search"
        android:title="@android:string/search_go"
        yourapp:showAsAction="ifRoom"  />
    <item android:id="@+id/action_settings"
        android:title="设置"
        yourapp:showAsAction="never"  />
</menu>
    ```

**二、 ContextMenu**
1. 方法执行流程：
    每次长按指定的View，自动在所按的位置处弹出menu。
2. 设置流程：
  * Activity中获取一个组件，如一个Button：
    `private Button btn;`
  * 让这个Button进行初始化并让其注册到ContextMenu中：
     ```
      btn  = (Button) findViewById(R.id.btn_actionbar);
      registerForContextMenu(btn);
     ```
3. 重写ContextMenu相关方法，让此方法触发时，弹出指定的ContextMenu【注意是ContextMenu类型的，不是Menu类型】：
   ```
@Override
public void onCreateContextMenu(ContextMenu menu, View v, ContextMenu.ContextMenuInfo menuInfo) {
    super.onCreateContextMenu(menu, v, menuInfo);
    MenuInflater mi = getMenuInflater();
    ///R.menu.activity_actionbar与上面OptionsMenu的一样!
    mi.inflate(R.menu.activity_actionbar,menu);
}
@Override
public boolean onContextItemSelected(MenuItem item) {
    return super.onContextItemSelected(item);
}
   ```

## ActionBar
---
1. ActionBar的基本使用：
    1. Android 版本 >= 3.0 ，所有使用Theme.Holo主题（及其子类）的Activity都包含了ActionBar，当`targetSdkVersion` 或 `minSdkVersion `属性被设置成“11”或者更大时，他是默认主题
    2. Android版本>=2.1 ，要添加ActionBar，需要加载Android Support库：`v7 appcompat`库（现在AS在新建项目时，基本都会为我们添加这个库）。然后：①让`Activity`继承`ActionBarActivity` ②在manifast.xml文件中，给<activity>添加一句：`<activity android:theme="@style/Theme.AppCompat.Light" ... >`

2. xml中设定每个Activity的ActionBar上显示的标题（Label）：
    通过设置清单文件中Activity标签中的label属性，如：`<activity android:name=".actionbar.MyActionBarActivity"    android:label="ActionBar活动" ……/>`
![](http://upload-images.jianshu.io/upload_images/2369895-3a277f9988102697.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. ActionBar从创建到行为实现
    * 创建相应的Activity：ActionBarActivity、AppCompatActivity（均位于v7 support库中，用于兼容到Android2.1版本，即API 7）
    * 创建Menu菜单：
      * 方式一：java代码
        在**onCreateOptionsMenu(Menu menu)**方法中，加入如下示例：
        ```
@Override
    public boolean onCreateOptionsMenu(Menu menu) {
  menu.add(Menu.NONE,ONE_ID,Menu.NONE,"1 Pixel");
        menu.add(Menu.NONE, TWO_ID, Menu.NONE, "2 Pixels");
        menu.add(Menu.NONE, EIGHT_ID, Menu.NONE, "8 Pixels");
       ……
        return super.onCreateOptionsMenu(menu);
    }
        ```
      * 方式二：xml定义后再引用
        先创建xml菜单布局文件**res/menu/my_menu.xml**：
        ```
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:yourapp="http://schemas.android.com/apk/res-auto" >
    <item android:id="@+id/action_search"
        android:icon="@android:drawable/ic_menu_search"
        android:title="@android:string/search_go"
        yourapp:showAsAction="ifRoom"
  />
    <item android:id="@+id/action_settings"
        android:title="设置"
        yourapp:showAsAction="never"
  />
…………
</menu>
        ```
          此时，ActionBar已经有了菜单列表了！
    * 完善ActionBar信息：
        在Java代码中完善ActionBar信息
        ```
getSupportActionBar().setIcon(R.mipmap.ic_launcher);
getSupportActionBar().setSubtitle("我是subTitle");
getSupportActionBar().setCustomView(R.layout.view_custom);
getSupportActionBar().setLogo(R.mipmap.ic_launcher);
getSupportActionBar().setElevation(50);
        ```
    * 最终截图
      ![](http://upload-images.jianshu.io/upload_images/2369895-e076b6dc075cb7c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. ActionBar的自定义背景
  完整示例：`res/values/themes.xml`
  ```
  <?xml version="1.0" encoding="utf-8"?>
<resources>  
  <!-- 应用于程序或者活动的主题 -->   
 <style name="CustomActionBarTheme"
      parent="@style/Theme.AppCompat.Light.DarkActionBar">  
      <item name="android:actionBarStyle">@style/MyActionBar</item>
       <item name="android:actionBarTabTextStyle">@style/MyActionBarTabText</item>
       <item name="android:actionMenuTextColor">@android:color/holo_blue_bright</item>
        <item name="android:windowActionBarOverlay">true</item>
       <!-- 支持库兼容 -->
       <item name="actionBarStyle">@style/MyActionBar</item>
       <item name="actionBarTabTextStyle">@style/MyActionBarTabText</item>
       <item name="actionMenuTextColor">@android:color/holo_blue_bright</item>
       <!--ActionBar的层次覆盖模式-->
        <item name="windowActionBarOverlay">true</item>
    </style>
   <!-- ActionBar 样式 -->
    <style name="MyActionBar"
       parent="@style/Widget.AppCompat.ActionBar">
       <item name="android:background">@android:color/holo_green_dark</item>
       <item name="android:titleTextStyle">@style/MyActionBarTitleText</item>
        <!-- 支持库兼容 -->
       <item name="background">@android:color/holo_green_dark</item>
        <item name="titleTextStyle">@style/MyActionBarTitleText</item>
    </style>
   <!-- ActionBar 标题文本 -->
    <style name="MyActionBarTitleText"
        parent="@style/TextAppearance.AppCompat.Widget.ActionBar.Title">
        <item name="android:textColor">@android:color/holo_red_light</item>
        <!-- 文本颜色属性textColor是可以配合支持库向后兼容的 -->
    </style>
    <!-- ActionBar Tab标签文本样式 -->
    <style name="MyActionBarTabText"
        parent="@style/Widget.AppCompat.ActionBar.TabText">
        <item name="android:textColor">@android:color/holo_blue_light</item>
        <!-- 文本颜色属性textColor是可以配合支持库向后兼容的 -->
    </style>
</resources>
  ```

4. 利用ActionBar快速构造返回上级的按钮
  截图：
  ![](http://upload-images.jianshu.io/upload_images/2369895-c87d204f1f047b1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  设置步骤：
    1. ActionBarActivity或者AppCompatActivity的onCreate()方法中设置：
      * `getSupportActionBar().setDisplayHomeAsUpEnabled(true);`  v7库兼容下使用
      * `getActionBar().setDisplayHomeAsUpEnabled(true);`  非兼容形式，minSdkVersion>=11时使用
    2. 在想要设置此功能的Activity的<activity>标签中设置如下：
      ```
        <activity android:name=".xxxx.SecondActivity"
    android:label="ActionBar活动"
    android:parentActivityName=".MyToolBarActivity"    >
    <!--meta-data用于兼容support4.0以下，指明上级活动-->
    <meta-data
        android:name="android.support.PARENT_ACTIVITY"
        android:value="com.xxxxx.xxxxx.MainActivity"
        />
</activity>
      ```

## Toolbar
---
**一、背景**
  * Android5.0推出的一个MD风格的导航栏。
  * 旨在取代ActionBar

**二、优势**
  * 比ActionBar更加灵活：
    * 设置导航栏图标
    * 设置App的logo
    * 支持设置标题和子标题
    * 支持添加一个或多个的自定义控件
    * 支持Action Menu
  * 可以放置任何位置

**三、用法**
  1. 导入**appcompat-v7** 的兼容包，使用 **android.support.v7.widget.Toolbar** 进行开发。
    ```
///基本格式
<android.support.v7.widget.Toolbar
         android:id="@+id/toolbar"
         android:layout_width="match_parent"
         android:layout_height="wrap_content"
    >
</android.support.v7.widget.Toolbar>
    ```
  2. 设置属性
    1. 方式一：xml中设置（* 注意：v7兼容包的缘故，toolbar的属性不在系统中而需要额外查找导入，所有不能用`android:xxx`，而需要自定义`xmlns:toolbar="http://schemas.android.com/apk/res-auto"`，另外，这种方式难以绑定menu菜单 *）
      ```
<android.support.v7.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="@color/colorAccent"
        toolbar:navigationIcon="@mipmap/ic_launcher"
        toolbar:logo="@mipmap/ic_launcher"
        toolbar:title="Title"
        toolbar:subtitle="subTitle"
        toolbar:titleTextColor="@color/colorPrimary"
        toolbar:subtitleTextColor="@color/colorPrimaryDark"
    >
    <!--引入普通View-->
    <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
           android:layout_margin="20dp"
           android:text="点击"
        />
    <!--引入自定义View-->
    <include
            layout="@layout/view_custom"/>
    </android.support.v7.widget.Toolbar>
      ```

      ![方式一截图](http://upload-images.jianshu.io/upload_images/2369895-395aa3bbd52008a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    2. 方式二：Java中设置
      ```
Toolbar toolbar  = (Toolbar) findViewById(R.id.toolbar);
toolbar.setNavigationIcon(R.mipmap.ic_launcher);///导航icon
toolbar.setLogo(android.R.drawable.ic_menu_search);///logo
toolbar.setTitle("Title");///标题
toolbar.setSubtitle("subTitle");///子标题
toolbar.addView(new Button(this));//可自己动态加CustomView
toolbar.inflateMenu(R.menu.activity_actionbar);///设置menu
      ```
       ![方式二截图](http://upload-images.jianshu.io/upload_images/2369895-c4e559b8db4a60ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  3. 扩展
    * 改变menu item的文字颜色
      * 首先，在themes.xml文件中添加一个<style>，注意parent为**parent="Theme.AppCompat.Light.NoActionBar"**
        ```
<style name="MyToolbarStyle" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="android:textColorPrimary">@android:color/holo_red_light</item>
</style>
        ```
      * 然后，给toolbar加上属性：** toolbar:popupTheme="@style/MyToolbarStyle"**
        ```
  <android.support.v7.widget.Toolbar
    android:id="@+id/toolbar"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@color/colorAccent"
    toolbar:popupTheme="@style/MyToolbarStyle"/>
        ```

       ![ 改变menu item的文字颜色截图](http://upload-images.jianshu.io/upload_images/2369895-e016baae7a52bd44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 感谢阅读
---
点击进入：[我的博客](http://androidjp.cn)
