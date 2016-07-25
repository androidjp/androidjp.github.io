---
title: Android开发细节--查漏补缺（二）：易忘难懂
date: 2016-07-25 14:59:00
categories:
- Android
tags:
- Android
- 面试
- 基础
---


> 这篇文章先汇总一下部分在平常容易遇到或可能遇到的一些难以理解或者容易忘记是什么的问题。

<!--more-->

---
1. android:supportsRtl="true"是什么？
  `android:supportsRtl="true"` 故名思义，“Rtl”就是“right to left” ，一般用于适配language文本方向，如切换系统语言为阿拉伯文时，actionbar布局要变为从右向左排列，就需要这个属性。
  > 注意：
此属性最低SDK版本为17（Android 4.2.x）
在使用这个属性时，尽量不适用`layout_marginRight`这类，而是使用`layout_marginEnd`这种。

  ---

2. Android res/menu/my_nemu.xml中不易记的属性：
  *  android:**alphabeticShortcut**="n" ：表示快捷键为：Ctrl+n 、Alt+n
  *  android:**orderInCategory** = "3"：表示在菜单列表中的排序，值可以为大于等于0的任何整数，数值越小，排序越前。
  * android:**checkableBehavior** = "single|all|none"：分别表示radio button、checkbox、checkable=false
 * <group android:id="xx"……> <item android:id="yy"/>……</group>： 用一个组，包含几个menu_item，这样，组内部的item的顺序就可以重新定义。
  ---

3. Android中用Java代码设置菜单项快捷键的方法：
  ```  
menu.setQwertyMode(true);
menu.findItem(TWO_ID).setNumericShortcut('2');
  ```
  ---

4. <RelativeLayout>中子view的属性：
  * **android:below="@+id/xx"** ：表示本view在xx的下面
  *  **android:above="@+id/xx"**：表示本view在xx的上面
  ---

5. implements Parcelable的对象的一般写法：（详见我的另一篇博文：[Android AIDL基础 -- Parcelable接口](http://www.jianshu.com/p/b1fcff2728bd)）
  ```
public class Student implements Parcelable{
    ///-------------------------------
    /// （1）：基本类型、String/CharSequence、Bundle、IBinder、Serializable、Parcelable 的成员
    private T t;
    ……
    ///---------------------------------
    ///（2）：构造方法（可选，怎么方便赋值成员变量值并构建实例，就怎么写）
    public Student() {
        ……
    }
   //=========================================================
   //  下面是Parcelable需要实现的方法
   //=========================================================
    protected Student(Parcel in) {
        ///建议：按成员变量顺序写【因为：需要与writeToParcel()方法中的写入顺序一致】
        id = in.readInt();//如：基本数据类型成员
        name = in.readString();//如：String类型成员
        home = in.readParcelable(Address.class.getClassLoader());//如：Parcelable类型成员
        ……
    }
    public static final Creator<Student> CREATOR = new Creator<Student>() {
        ///CREATOR成员，格式必须如下，‘CREATOR’变量名也固定。
        @Override
        public Student createFromParcel(Parcel in) {
            return new Student(in);
        }
        @Override
        public Student[] newArray(int size) {
            return new Student[size];
        }
    };
    @Override
    public int describeContents() {
        /// 描述标志，默认为0即可
        return 0;
    }
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        ////写入Parcel的方法【需要与writeToParcel()方法中的写入顺序一致】
        dest.writeInt(id);
        dest.writeString(name);
        dest.writeParcelable(home, flags);
        ……
    }
}
  ```
6. Android Studio获取Android签名证书SHA1的简易步骤：
  * Android Studio中打开terminal或者配置了全局变量之后直接打开终端输入：
    **`keytool -list -v -keystore 你的签名文件所在的路径`**
  * 输入密钥库密码
  * 成功显示：MD5、SHA1、SHA256等信息
    > 想知道Eclipse的获取方式，更详细的地址可以看：[SHA1查看](http://lbs.amap.com/dev/ticket#/faq/86)
  ---
7. JSON封装库的效率对比：
  如果对JSON的解析速度敏感：大文件用：**Jackson**，小文件用：**Gson**，两种文件都有用：**JSON.simple**

  ---
8. Java代码执行new一个类实例对象的过程中，static块、static方法、main方法的执行顺序：
  * static块【最先执行，因为静态代码块属于类层次，在类编译时就执行】
  *  -> main方法
  *  -> 成员块
  *  -> 构造方法
  *  -> static方法【在被调用时才执行】
  > 详细可以看看本人的相关文章：[Java面试相关（一）-- Java类加载全过程](http://www.jianshu.com/p/ace2aa692f96)
  ---
9. 在写Fragment的时候，Fragment.onCreateView()方法中，一般是这样的一个结构：
  ```
@Nullable
@Override
public View onCreateView(LayoutInflater inflater,  ViewGroup container,  Bundle savedInstanceState) {
  if (rootLayout == null) {
    rootLayout = inflater.inflate(R.layout.fragment_reclist, null);///①
    rootLayout = inflater.inflate(R.layout.fragment_reclist, container, false);///②
    ///rootLayout = inflater.inflate(R.layout.fragment_reclist, container);///③这样不行！！
    ……  
  }
……
return rootLayout;
}
  ```
这里，要注意代码中注释掉的地方，这里有一个对于Fragment的知识点：
 `inflater.inflate(id, ViewGroup);`
说明：
 这个方法传入：【目的布局的R.layout 的id】 + 【目的布局的父容器】。那么，对于Fragment的来说，Fragment的onCreateView()方式最终返回的View会自动判断本Fragment所在的父容器位置。
 **为什么上面代码③不行呢?** 答：因为`inflater.inflate(id, ViewGroup);`最终会调用`inflater.inflate(id, ViewGroup,boolean);` 方法，当ViewGroup不为null时，这方法的第三个参数值会为true，表示的是‘inflate出来的布局自动加到它的父viewgroup中’，那么问题就来了，如果这个父布局是一个ViewPager或者其他不易确定子View放置位置的容器，那么，方法就会**报错**，因为找不到插入点！所以，对于Fragment的布局，一般情况不需要设置自动添加到父布局，所以，上述代码①和②二选一都可以。

  ---
10. Fragment 与 它所在的Activity的生命周期方法执行次序：
    下面用一个常用布局作为示例【FragmentActivity中用ViewPager装载两个Fragment 部分代码示例】来演示 ，看运行的Log就可以说明整个执行顺序：
  ```
public class CommonTabActivity extends FragmentActivity {
……
/// TabLayout + ViewPager的主界面容器
PagerAdapter mPagerAdapter;
ViewPager mViewPager;
TabLayout mTabLayout;
////两个 Fragment
RecListFragment homeFragment;
RecListFragment collectionFragment;
……
 @Override
 protected void onCreate(Bundle savedInstanceState) {
     super.onCreate(savedInstanceState);
     setContentView(R.layout.activity_commontab);
     initView();
 }
  private void initView() {
    ///初始化ViewPager等控件
    ……

        mPagerAdapter = new FragmentPagerAdapter(getSupportFragmentManager()) {
            @Override
            public Fragment getItem(int position) {
                switch (position) {
/////////////////////////使用第一个Fragment
                    case 0:
                    default:  
                        if(homeFragment==null){
                            homeFragment = new RecListFragment();
                        }
                        return homeFragment;
///////////////////////////使用第二个Fragment
                    case 1:
                        if (collectionFragment==null){
                            collectionFragment =  new RecListFragment();
                        }
                        return collectionFragment;
                }
            }
            …………
        };
        mViewPager.setAdapter(mPagerAdapter);
        mTabLayout.setupWithViewPager(mViewPager);
  }
}
  ```
  上面的结构大家应该都比较熟悉，那么，我测试后给大家列出几种情况下Activity和两个Fragment的生命周期方法执行情况：
  *  本Activity被startActivity()等方式启动：
    ![](http://upload-images.jianshu.io/upload_images/2369895-54b2a04304e2f186.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * 本Activity由于点击HOME、MENU等弹回桌面时：
     下图是Android N版本下点击MENU按键时的操作截图（类似平常我们长按HOME键弹出运行中程序）：
    ![](http://upload-images.jianshu.io/upload_images/2369895-c108a164f086f0f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    ![](http://upload-images.jianshu.io/upload_images/2369895-71511813a835a544.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * 本Activity启动其他Activity，并被完全遮掩覆盖时：
    ![](http://upload-images.jianshu.io/upload_images/2369895-cd5b8e9ce86970d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * Activity退出时：
    ![](http://upload-images.jianshu.io/upload_images/2369895-9a0e6a728886c2d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * 总结：
      * 启动Activity时，由于ViewPager默认采用预加载的方式，所以，虽然默认显示第一个Fragment，第二个Fragment依然会被加载和初始化。
      * 启动Activity时，Activity先执行了它本身的onCreate()->onStart()->onResume()，之后，ViewPager才真的开始运行起来并加载指定给他的Fragment，Fragment才开始它的生命周期。
      * 不论Activity是被另一个Activity覆盖、还是App被返回桌面、或者是弹出启动中应用的界面遮掩Activity，此时，Activity都是被完全遮掩（弹出Dialog等不全屏的窗体时，Activity属于部分被遮掩），此时，Activity都会调用到onStop()方法，并且这种方式中Fragment与Activity的生命周期方法执行顺序都一样(如上面截图)。
      * 当Activity退出时，①Fragment执行onStop()后，Activity再执行onStop() ②Fragment执行onDestroyView()和onDestroy()后，Activity才执行onDestroy()
  * 另外，如果到这里，忘了Fragment整个生命周期是怎么样的，这里，本人附上***<u>Fragment的生命周期图解</u>***：
  ![Fragment的生命周期](http://upload-images.jianshu.io/upload_images/2369895-3cf75cc2042a2e15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  ---

### 感谢阅读～
喜欢的读者可以点个关注，本人将持续发布Android相关文章～
