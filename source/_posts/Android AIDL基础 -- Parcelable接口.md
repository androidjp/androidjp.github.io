---
title: Android AIDL基础 -- Parcelable接口
date: 2016-07-22 20:17:24
categories:
- Android
- AIDL
tags:
- Android
- AIDL
- Parcelable
---
## 什么是Parcelable接口
---
* **作用：**给类设计的接口，类一旦实现这个接口，该类的对象即可进行一个“从对象存储为Parcel ， 从Parcel读取为对象”的一个读写过程。
* **目的：**实现序列化
* **原因：**
    序列化，为了：
    1） 永久保存对象，保存对象的字节到本地文件中
    2） 通过序列化对象在网络中传递对象
    3） 通过序列化在进程间传递对象
<!--more-->
* **与Serializable对比：**
  1）在使用Intent传递数据时：
    * Serializable: Bundle.putSerializable(Key, *实现了Serializable的对象*);
    * Parcelable: Bundle.putParcelable(Key, *实现了Parcelable的对象*);

  2）内存使用性能上：
    * Parcelable > Serializable
    > 由于Serializable在序列化时会产生大量的临时变量，从而引起频繁的GC。   

  3）数据存储在磁盘上的情况：
    * Serialzable > Parcelable
    > Parcelable不能使用在要将数据存储在磁盘上的情况，因为Parcelable不能很好的保证数据的持续性在外界有变化的情况下。尽管Serializable效率低点，但此时还是建议使用Serializable
* 支持的成员数据类型：
  1） 普通数据类型【如：int、double等以及他们的封装类如Integer、Float等】以及他们的数组类型【如：int[]、double[]等】***注意：不支持：List[Integer]类型等List类型***
  2） String/CharSequence及其数组String[]、ArrayList[String]
  3） Bundle 和 List<Bundle>
  4） Serializable ***注意：不支持List<Serializable>***
  5） Binder 和 List<Binder> ，IBinder 和 List<IBinder>
  6） Parcelable实现类
  > 重温：Intent/Bundle传递支持的数据类型有：
    * 基本数据类型和它的数组类型
    * String/CharSequence和它的数组类型
    * Serializable
    * Parcelable


## 实现Parcelable接口
---
* **实现步骤**
  1. `implements Parcelable`
  2. 重写`writeToParcel`方法，将你的对象序列化为一个Parcel对象，即：将类的数据写入外部提供的Parcel中，打包需要传递的数据到Parcel容器保存，以便从 Parcel容器获取数据。
  3. 重写`describeContents`方法，内容接口描述，默认返回0就可以。
  4. 实例化静态内部对象`CREATOR`实现接口`Parcelable.Creator`

* **没有Parcelable成员的Parcelable接口实现示例**：
  ```
/** * 测试：地址
 * Created by androidjp on 16-7-22.
 */
public class Address implements Parcelable{
    ///（可以）一般数据类型和String，如：String、int、Integer、double、Float等
    public String country;
    public String city;
    public String street;
    ///（可以）Bindle类型【由于Bindle本身实现了Parcelable】
    public Bundle bundle;
    ///（可以）Serializable类型
    public Serializable serializable;
    ///（可以）Binder类型
    public Binder binder;
    public Integer i;
    public int j;
    public List<String> stringList;
    public List<Bundle> bundleList;
//    public List<Serializable> serializableList;
    public ArrayList<IBinder> binderList;
//    public List<Integer> integerList;
    public int[] ints;

    public Address(String country, String city, String street) {
        this.country = country;
        this.city = city;
        this.street = street;
    }
   //=========================================================
   //  下面是Parcelable需要实现的方法
   //=========================================================
    protected Address(Parcel in) {
        country = in.readString();
        city = in.readString();
        street = in.readString();
        bundle = in.readBundle();
        serializable = in.readSerializable();
        binder = (Binder) in.readStrongBinder();
        i = in.readInt();
        j = in.readInt();
        stringList = in.createStringArrayList();
        bundleList = in.createTypedArrayList(Bundle.CREATOR);
        binderList = in.createBinderArrayList();
        ints = in.createIntArray();
    }
    /*
    * 读取接口，目的是要从Parcel中构造一个实现了Parcelable的类的实例处理。
    * 因为实现类在这里还是不可知的，所以需要用到模板的方式，继承类名通过模板参数传入。
    * 为了能够实现模板参数的传入，这里定义Creator嵌入接口,内含两个接口函数分别返回单个和多个继承类实例。
    */
    public static final Creator<Address> CREATOR = new Creator<Address>() {
        @Override
        public Address createFromParcel(Parcel in) {
            return new Address(in);
        }
        @Override
        public Address[] newArray(int size) {
            return new Address[size];
        }
    };

   /////内容描述接口，基本不用管
    @Override
    public int describeContents() {
        return 0;
    }
   //写入接口函数，打包
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(country);
        dest.writeString(city);
        dest.writeString(street);
        dest.writeBundle(bundle);
        dest.writeSerializable(serializable);
        dest.writeStrongBinder(binder);
        dest.writeInt(i);dest.writeInt(j);
        dest.writeStringList(stringList);
        dest.writeTypedList(bundleList);
        dest.writeBinderList(binderList);
        dest.writeIntArray(ints);
    }
}
  ```
* **加上Parcelable成员的Parcelable接口实现示例**：
  ```
/**
 * 测试：学生
 */
public class Student implements Parcelable{
    private int id;
    private String name;
    private Address home;////地址成员变量（已经实现了Parcelable接口）
    public Student(int id, String name, Address home) {
        this.id = id;
        this.name = name;
        this.home = home;
    }
   //=========================================================
   //  下面是Parcelable需要实现的方法
   //=========================================================
    protected Student(Parcel in) {
        id = in.readInt();
        name = in.readString();
        home = in.readParcelable(Address.class.getClassLoader());
    }
    public static final Creator<Student> CREATOR = new Creator<Student>() {
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
        return 0;
    }
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(name);
        dest.writeParcelable(home, flags);
    }
}
  ```

## 扩展
---
1. 由于Parcelable对象不能向Serializable那样将对象保存到文件中等持久化操作，那么，我的对象要怎么做？
    答： `public class Student implements Parcelable,Serializable{……`，让类同时实现两个接口，即可使他能够序列化并存储到文件中。

2. 什么是Parcel？
    答：简单来说，Parcel就是一个存放数据的容器。Android中以Binder机制方式实现来IPC，就是使用了Parcel来进行Client和Server间的数据交互，而且AIDL的数据也是通过Parcel来交互的。同样的，在Java中和C/C++中，都有Parcel的实现【Parcel在C/C++中，直接使用内存来读取数据，所以此时Parcel它更加快速】
    换句话理解Parcel：我们知道，类A和类B可能想要通信，那么，要进行交流，A肯定不想把自己的实例（包括成员变量、方法等）整个复制到B那边，并且，A和B在同一个线程、甚至同个进程的不同线程都好说，如果是不同进程呢？那得怎么传，通过网络之类的咯？所以，就有了Parcel这个“打包”一说，A把A的一些信息进行说明，将这些说明打包（而不同打包自己的具体东西），然后把信息传给B，B读了之后，根据A给的提示选择，将选择同样用Parcel打包传回给A，A收到就跑，跑完数据后，返回结果又同样Parcel装着给到B，整个通信过程类似这样。
