---
title: 一篇文章带Go小白入门
date: 2020-02-25 22:46:41
tags:
- Go
categories:
- Go
---
> 听说Go语言很火，什么大方向都能扯到他，什么区块链、大数据、大型高性能Web应用等，特地整理一波Go相关背景与基础知识，分享给大家。

<!--more-->

# Go简介
## 定义
* 维基百科：Go，又称为Golang（Go Language）。Google开发的一种静态强类型、编译型、并发型、具有垃圾回收功能的类C编程语言。

## 历史
* 诞生时间：2007年末。
* 开发者：Robert Griesemer, Rob Pike, Ken Thompson 设计Go并主持开发，后来加入了Ian Lance Taylor, Russ Cox等。(Pike听完C++0x的演讲，回到办公室，开始编译C++，编译过程中，Pike转身和Robert讨论语言问题，拉上Ken爷爷一起合计，群嘲C++的一些沙雕设计后，没等编译完成，三人一拍即合，搞了Go语言出来。GO语言之父Pike提到：GO语言是以C为原型，以C++为目标而设计的，希望C++程序员能以GO作为替代品。因为他觉得C++忒复杂了，要解救程序员于水火。)
* 开源时间：2009-19
* 刚开始的Go：在Linux及Mac OS X平台上进行了实现，后来追加了Windows系统下的实现。
* Go 1.0 稳定版发布时间：2012

### Go核心团队
各个大牛：
![](/images/20200225/go_da_niu.png)

核心设计师Pike和Ken都是出身自贝尔实验室，Ken之于Pike，亦师亦友，共同发明了UTF-8，还基情四射地结对编程过，感情好的穿一条裤子。

Pike是Unix先驱，贝尔实验室最早跟Ken、Dennis一起开发Unix的猛人，Plan9 OS的灵魂人物。大胡子Ken爷爷则是Unix之父，和Dennis一起发明了C语言，殿堂骨灰级程序员，早已是名满天下。

技术实力毋容置疑，不过这哥俩都是玩Kernel的，经历相同，理念相近，分歧会比较少，他们也都坦承C用得最多最熟，所以注定了GO的类C特性，不过这会不会导致GO设计上的思维火花不足，对OOP以及现代编程思想的支持不足，亦未可知。

## Go 的哲学
* **少即是多**：Less Is More，大道至简；
* **世界是并行的**：世间万物是并行发生的；
* **世界是物质组成的**：微观世界由小的粒子组合成大的粒子；宏观世界由小的物体组合成大的物体；
   * 继承只能描述现实世界的一小部分，使用继承是不全面的；
   * Go的设计选择的是组合，这个和现实世界比较吻合的设计，表现力更强；
* **世界是标准化的**：硬件是标准化的，软件也应如此，Go的接口是DUCK模型，接口是非侵入式的；
* **正交性**：Go的多个特性都是正交性的，正交性是保持事物稳定和简单的最好设计；
* **二八定律**：80%代码只使用20%特性，增加语言特性，并不能提升效率，反而会增加复杂性，提高犯错率，加重程序员心智负担；
* **统一格式化**：给你（我认为）最好的，而不是给你选择。就像iPhone一样，用户太笨了，他们根本不知道自己需要什么，就让帮主替你安排好一切吧。当然，因为规则不是大家定的，而已某一个人定的，那就难免有些主观，规则好坏，见仁见智。

## 特点与特性
* Go是基于Inferno操作系统所开发的；
* Go每半年发布一个二级版本（即从a.x升级到a.y）；
* 原生并发，以东尼·霍尔的通信顺序进程（CSP）为基础的goroutine，适合现代多核机器；
* 语法简单清晰；
* 强编码规则；
* 开源；
* 高效的垃圾回收机制；
* 强大的标准库，对网络编程等的良好支持；
* 内存管理、数组安全、编译迅速；
* 语法接近C语言，但其内部运行机制又偏向Java；
* CGO提供了GO调用C机制，扩展了GO的能力边界；
* 内嵌关联数组；
* 非侵入式的接口设计；

## Go vs C/C++
![](/images/20200225/go_1.png)

## 性能对比
虽然Go号称兼备C++的运行效率和PHP的开发效率，但从benchmarks数据上看，Go的运行效率还不能真的干趴老大哥Java。
![](/images/20200225/go_2.png)

当然，Go再加以优化，性能应会更强。


## 适用场景
> Go 语言被设计成一门应用于搭载 Web 服务器，存储集群或类似用途的巨型中央服务器的系统编程语言。

* 高性能分布式系统领域
* 游戏服务端

# 安装与配置
IDE：JetBrains GoLand 2019.2.3

## Windows步骤
1. 去[官网](https://golang.org)下载exe，双击安装（默认安装目录：`C:\Go`）
2. 添加`c:\Go\bin`到环境变量Path 【其实这一步exe已经帮你做了】
3. 输入命令进入验证
    ```
    go version
    ```

## Linux步骤 (MacOS参考他们俩就好了)
1. 下载tar.gz包
2. 解压（前提：保证你有root权限）
```
// 解压到安装目录：/usr/local/go
tar -C /usr/local go1.12.4.linux-amd64.tar.gz
```
3. 设置环境变量
    ```
    export GOROOT = /usr/local/go
    export PATH = $PATH:$GOROOT/bin
    ```
  * 对所有用户有效：`/etc/profile`
    ```
    source ~/.profile
    ```
  * 对当前用户生效：`$HOME/.profile`
    ```
    source /etc/profile
    ```

# Go特色

## 工作区目录结构
```
src // 引用的外部库
pkg // 编译时，生成的对象文件
bin // 编译后的程序
```

## Goroutine
* 定义：Go的主要功能之一，简单易用的并行设计。
* 作用：让程序以异步的方式运行，类似JS的`async await`，为了解决一个函数导致主线程中断的问题。
* 原理：Goroutine 是类似线程的概念（并不是线程，类似于协程，一种轻量级线程）。线程属于系统层面，通常来说创建和销毁一个线程需要消耗较多资源且不易管理。又称为“并发”，一个Go程序可以运行超过数万个Goroutine，并且性能都是原生级别的，随时能够关闭。一个核心里可以有多个Goroutine，可通过`GOMAXPROCS`参数限制其占用系统线程的数目。
* 例子：
    ```
    func main() {
        // 透過 `go`，我們可以把這個函式同步執行，
        // 如此一來這個函式就不會阻塞主程式的執行。
        go loop()
    	// 執行另一個迴圈。
    	loop()
    }
    ```

## 编译器
* 当前有两个Go编译器分支，分别为官方编译器gc和gccgo。官方编译器在初期使用C写成，后用Go重写。Gccgo是一个使用标准GCC作为后端的Go编译器。
* 官方编译器支持跨平台编译（但不支持CGO），允许将源码编译为可在目标系统、架构上执行的二进制文件。

# 环境相关：GOROOT 和 GOPATH
* `GOROOT`:  安装 go 的路径
* `GOPATH`:  是我们定义的自己的工作空间

### 手动配置GOROOT
如果是默认一路next，是会自动配置好`GOROOT`系统环境变量的。

1. 首先，在环境变量中添加`GOROOT`，值为 go 安装目录；
2. 然后在环境变量 `PATH` 中添加 go 安装目录下的 `bin` 文件夹。

### 手动添加GOPATH
接着添加一个环境变量 `GOPATH`，值为你自己希望的工作目录：

配置完毕后，记得重启cmd工具，然后输入`go env`即可看到配置内容。


![Mr1LDA.png](https://s2.ax1x.com/2019/11/17/Mr1LDA.png)

# 如何运行.go文件
Idea在create一个`new simple Application`时，会自动生成一个`main()`，然后，即可被Go编译器识别并运行。

给你一个hello world代码：
```
package main

import "fmt"

func main() {
	fmt.Println("Hello world")
}
```

运行方式一：
1. run，直接运行
    ```
    go run helloWorld.go
    ```

运行方式二：
1. build一下，得到二进制文件(`.exe`)
    ```
    go build helloWorld.go
    ```
2. 然后，执行二进制文件
    ```
    helloWorld
    ```

# Go中的函数定义
看到`fmt.Println("Hello world")`，我们会在想：为什么Go会用首字母大写去定义一个函数名？

当标识符（包括常量、变量、类型、函数名、结构字段等等）以一个大写字母开头，如：Group1，那么使用这种形式的标识符的对象就可以被外部包的代码所使用（客户端程序需要先导入这个包），这被称为导出（像面向对象语言中的 public）；标识符如果以小写字母开头，则对包外是不可见的，但是他们在整个包的内部是可见并且可用的（像面向对象语言中的 protected ）。

# Go 关键字一览
25个关键字：
```
break	default	func	interface	select
case	defer	go	map	struct
chan	else	goto	package	switch
const	fallthrough	if	range	type
continue	for	import	return	var
```
36 个预定义标识符：
```
append	bool	byte	cap  	close	complex 	complex64	complex128	uint16
copy	false	float32	float64	    imag	int	 int8	int16	uint32
int32	int64	iota	len	make	new	nil	panic	uint64
print	println	real	recover  	string	true	uint	uint8	uintptr
```


# Go 数据类型 注意点
### 布尔型
布尔型的值只可以是常量 true 或者 false。一个简单的例子：var b bool = true。

### 数字类型
整型 int 和浮点型 float32、float64，Go 语言支持整型和浮点型数字，并且支持复数，其中位的运算采用补码。

类型 | 描述
--- | ---
uint8 | 无符号 8 位整型 (0 到 255)
uint16 |无符号 16 位整型 (0 到 65535)
uint32 |无符号 32 位整型 (0 到 4294967295)
uint64 |无符号 64 位整型 (0 到 18446744073709551615)
int8 |有符号 8 位整型 (-128 到 127)
int16 |有符号 16 位整型 (-32768 到 32767)
int32 |有符号 32 位整型 (-2147483648 到 2147483647)
int64 |有符号 64 位整型 (-9223372036854775808 到 9223372036854775807)

### 浮点型
类型 | 描述
--- | ---
float32 | IEEE-754 32位浮点型数
float64 | IEEE-754 64位浮点型数
complex64 | 32 位实数和虚数
complex128 | 64 位实数和虚数

### 其他数字类型
类型 | 描述
--- | ---
byte | 类似 uint8
rune | 类似 int32
uint | 32 或 64 位
int | 与 uint 一样大小
uintptr | 无符号整型，用于存放一个指针

### 字符串类型
字符串就是一串固定长度的字符连接起来的字符序列。Go 的字符串是由单个字节连接起来的。Go 语言的字符串的字节使用 UTF-8 编码标识 Unicode 文本。

### 派生类型
包括：
* (a) 指针类型（Pointer）
* (b) 数组类型
* (c) 结构化类型(struct)
* (d) Channel 类型
* (e) 函数类型
* (f) 切片类型
* (g) 接口类型（interface）
* (h) Map 类型

# Go 语言变量
## 变量声明
声明格式： `var identifier type`

可以一次声明多个变量：`var identifier1,identifier2 type`

实例：
```
package main
import "fmt"
func main() {
    var a string = "xxxx"
    fmt.Println(a)
    var b,c int = 1,2
    fmt.Println(b, c)
}
```

### 第一种：指定变量类型，如果没有初始化，则变量默认为零值。
```
var v_name v_type  // 声明，此时 v_name 为零值
v_name = value     // 赋值
```
> 零值：变量没有做初始化时系统默认设置的值。

如：
```
// 声明一个变量并初始化
var a1 = "Jay"
fmt.Println(a1) // Jay

// 没有初始化就为零值
var b1 int
fmt.Println(b1) // 0

// bool 零值为 false
var c1 bool
fmt.Println(c1) // false
```

**零值分类**：
* 数值类型（包括complex64/128）为 0
* 布尔类型为 false
* 字符串为 ""（空字符串）
* 以下几种类型为 nil：
    ```
    var a *int
    var a []int
    var a map[string] int
    var a chan int
    var a func(string) int
    var a error // error 是接口
    ```

如：
```
package main

import "fmt"

func main() {
    var i int     // 0
    var f float64 // 0
    var b bool    // false
    var s string  // ""
    fmt.Printf("%v %v %v %q\n", i, f, b, s) // 0 0 false ""
}
```

### 第二种：根据值自动判定对象类型
```
var isValid = true
fmt.Println(isValid) // true
```

### 第三种：`:=`
`isValid := true` 相当于 `var isValid bool = true`。 所以，一定要是新声明变量，才能用这个`:=`


### 值类型与引用类型

#### 1. 值类型
所有像 `int`、`float`、`bool` 和 **`string`** 这些基本类型都属于值类型，使用这些类型的变量直接指向存在内存中的值：
![MrUsOO.png](https://s2.ax1x.com/2019/11/17/MrUsOO.png)
```
var i = 7
var j = i // 实际上是在内存中将 i 的值进行了拷贝
fmt.Println(&i, &j) // 0xc0000580d8 0xc0000580e0
// 发现两个变量指向的内存地址的位置不同
```
![MrURkd.png](https://s2.ax1x.com/2019/11/17/MrURkd.png)

#### 2. 引用类型
一个引用类型的变量 r1 存储的是 r1 的值所在的内存地址（数字），或内存地址中第一个字所在的位置。
![MralAH.png](https://s2.ax1x.com/2019/11/17/MralAH.png)
这个内存地址为称之为指针，这个指针实际上也被存在另外的某一个字中。

同一个引用类型的指针指向的多个字可以是在连续的内存地址中（内存布局是连续的），这也是计算效率最高的一种存储形式；也可以将这些字分散存放在内存中，每个字都指示了下一个字所在的内存地址。

当使用赋值语句 r2 = r1 时，只有引用（地址）被复制。

如果 r1 的值被改变了，那么这个值的所有引用都会指向被修改后的内容，在这个例子中，r2 也会受到影响。

# Go 常量

### 一般玩法：const
常量定义格式：
```
const identifier [type] = value
```
```
const LENGTH int = 10
const WIDTH int = 5
var area int
const a, b, c = 1, false, "str" //多重赋值

area = LENGTH * WIDTH
fmt.Printf("面积为 : %d", area) // 面积为：50
println()
println(a, b, c) // 1 false str
```
常量可以用`len()`, `cap()`, `unsafe.Sizeof()` 函数计算表达式的值。常量表达式中，函数必须是内置函数，否则编译不过：
```
import (
	"fmt"
	"unsafe"
)
.........
const (
	a_1         = "abc"
	a_len       = len(a_1)
	a_type_size = unsafe.Sizeof(a_1)
)
println(a_1, a_len, a_type_size) // abc 3 16
```

### iota
iota，特殊常量，可以认为是一个**可以被编译器修改的常量**。

iota 在 `const` 关键字出现时将被重置为 `0` (const 内部的第一行之前)，`const` 中每新增一行常量声明将使 iota 计数一次 (<u>iota 可理解为 const 语句块中的行索引</u>)。

iota 可以被用作枚举值：
```
const (
    a = iota // 0
    b = iota // 1
    c = iota // 2
)
```
第一个 iota 等于 0，每当 iota 在新的一行被使用时，它的值都会自动加 1；所以 a=0, b=1, c=2 可以简写为如下形式：
```
const (
    a = iota  // 0
    b         // 1
    c         // 2
)
```
iota用法例子：
```
const (
	a = iota   //0
	b          //1
	c          //2
	d = "ha"   //独立值，iota += 1
	e          //"ha"   iota += 1
	f = 100    //iota +=1
	g          //100  iota +=1
	h = iota   //7,恢复计数
	i          //8
)
fmt.Println(a,b,c,d,e,f,g,h,i)
// 0 1 2 ha ha 100 100 7 8
```
有趣例子二--左移：
```
package main

import "fmt"
const (
    i=1<<iota // 1<<0 = 1
    j=3<<iota // 3<<1 = 6
    k         // 3<<2 = 12
    l         // 3<<3 = 24
)

func main() {
    fmt.Println("i=",i) // 1
    fmt.Println("j=",j) // 6
    fmt.Println("k=",k) // 12
    fmt.Println("l=",l) // 24
}
```

# 运算符
优先级 | 运算符
--- | ---
5| `*`, `/`, `%`, `<<`, `>>`, `&`, `&^`
4| `+`, `-`, `|`, `^`
3| `==`, `!=`, `<`, `<=`, `>`, `>=`
2| `&&`
1| `||`

特别说明一下：
* `&`：返回变量存储地址（`&a;` 将给出变量的实际地址）
* `*`：指针变量。（`*a;` 是一个指针变量）

```
var x uint8 = 3
var y uint8 = 2
fmt.Println(x >> y)// 0
xStr := &x
fmt.Println(xStr)  // 0xc00000a0b0
fmt.Println(*xStr) // 3
```

# 条件语句
## Type Switch
switch 语句还可以被用于 type-switch 来判断某个 interface 变量中实际存储的变量类型。

例子：
```
package main

import "fmt"

func main() {
   var x interface{}
     
   switch i := x.(type) {
      case nil:   
         fmt.Printf(" x 的类型 :%T",i)                
      case int:   
         fmt.Printf("x 是 int 型")                       
      case float64:
         fmt.Printf("x 是 float64 型")           
      case func(int) float64:
         fmt.Printf("x 是 func(int) 型")                      
      case bool, string:
         fmt.Printf("x 是 bool 或 string 型" )       
      default:
         fmt.Printf("未知型")     
   }   
}
```
执行结果：
```
x 的类型 :<nil>
```
##  fallthrough
使用 fallthrough 会强制执行后面的 case 语句，fallthrough 不会判断下一条 case 的表达式结果是否为 true。
```
func main() {
	// 条件语句
	a := 2
	switch {
	case a == 1:
		fmt.Println("1、case 条件语句为 false")
		fallthrough
	case a == 2:
		fmt.Println("2、case 条件语句为 true")
		fallthrough
	case a == 3:
		fmt.Println("3、case 条件语句为 false")
		fallthrough
	case a == 4:
		fmt.Println("4、case 条件语句为 true")
	case a == 5:
		fmt.Println("5、case 条件语句为 false")
		fallthrough
	default:
		fmt.Println("6、默认 case")
	}
}
```
结果：
```
2、case 条件语句为 true
3、case 条件语句为 false
4、case 条件语句为 true
```

# 循环语句
## goto
例子：在变量 a 等于 15 的时候跳过本次循环并回到循环的开始语句 LOOP 处
```
func main() {
	var a int = 10
LOOP:
	for a < 20 {
		if a == 15 {
			a += 1
			goto LOOP
		}
		fmt.Println(a)
		a++
	}
}
```
结果：
```
10
11
12
13
14
16
17
18
19
```

# 指针
```
func main() {
	var x uint8 = 3
	var y uint8 = 2
	fmt.Println(x >> y)// 0
	// 获取x 的指针
	xStr := &x
	fmt.Println(xStr)  // 打印指针的值（也就是x值的内存地址）： 0xc00000a0b0
	fmt.Println(*xStr) // 3

	var a int = 20
	var ip *int
	ip = &a
	println("a 值", a) // 20
	println("a 变量地址", &a) // a 变量地址 0xc000081f40
	println("ip 变量的值", ip)// ip 变量的值 0xc000081f40
	println("通过 （指针）*ip 返回值", *ip) // 通过 （指针）*ip 返回值 20

	println("空指针")
	var nilIp *int
	println("默认指针为空，值为", nilIp) // 默认指针为空，值为 0x0
	println("到底是不是空指针的判断：", "if(nilIp == nil)", "结果是", nilIp == nil) // true
}
```

# 函数
```
func main() {
	println("printInt:", printInt())
	println("add(1,2):", add(1, 2))
	s := "hello world"
	println("s的长度:", len(s))

	baseSalary := 8000.0
	var rate float32 = 1.2
	var res float64 = calculateFinalSalary(baseSalary, rate)
	fmt.Printf("原来工资为%f,晋升后的工资为：%f\n", baseSalary, res)

	x1 := "Mike"
	x2 := "David"
	res1, res2 := swap(x1, x2)
	fmt.Printf("返回多个值的函数：原来是%s,%s, 交换位置后得到%s, %s\n", x1, x2, res1, res2)

	println()
	println()

	println("看看Go语言 string是否是值传递的：")
	mike :="Mike"
	playSwitch(mike)
	changePlayer(mike, "Louis")
	playSwitch(mike)
}

func changePlayer(name string, newName string) {
	println(name, "已经换人了，现在是", newName)
	name = newName
}

func playSwitch(player string) {
	println(player, "playing the Switch ~")
}

func swap(x1 string, x2 string) (string, string) {
	return x2, x1;
}

func calculateFinalSalary(base float64, promoteRate float32) float64 {
	return base * float64(promoteRate)
}

func add(a int, b int) int {
	return a + b
}

func printInt() int {
	return 1
}
```
注意：
* Go中的String的确是值传递的；
* Go的函数可以返回多个参数；
* 进行运算时，类型不一致需要显式进行类型转换；

# 变量作用域
Go也有全局变量和局部变量的概念：
```
package main

// 全局变量
var g string = "ABC"
var x string = "ABC"

func main() {
	// 局部变量
	var x string = "sss"

	println(g, x) // ABC sss
}
```

# 数组
Go语言中的数组有一些特点：
* 数组一旦初始化，那就是固定大小（和Java的 int[] 等一个道理）；
* 只有相同长度和类型的数组之间，才能直接进行赋值，否则，需要遍历元素来赋值；

```
package main
import "fmt"
func main() {
	//	声明数组
	var balance [10] float32
	balance[0] = 12
	println("balance数组长度", len(balance)) // 10
	println("balance数组cap", cap(balance))  // 10
	println("balance数组指针", &balance)     // 0xc00006beb0

	var actors [10] string
	actors[0] = "XiaoMing"
	println("actors数组长度", len(actors))   // 10
	println("actors数组cap", cap(actors))    // 10
	println("actors数组指针", &actors)       // 0xc00006bee8

	// 初始化数组(括号里的参数个数不得大于 给定的数组长度)
	// 报错：因为前面已经声明 balance 为长度是 10 的float32 数组了
	// balance = [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
	var arr1 = [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
	println(len(arr1))                       // 5
	// 赋值的数组形状一样，所以不会报错
	// 这里可以不指定数组长度，默认按照括号中的元素个数来设置数组大小
	arr1 = [...]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
	println(len(arr1))                       // 5

	println("遍历数组（保留2位小数）：")
	for i := 0; i < len(arr1); i++ {
		fmt.Printf("%.2f,", arr1[i])         // 1000.00,2.00,3.40,7.00,50.00,
	}
}
```
# 结构体
```
// 结构体
// Go 当中，数组可以存储同一类型的数据，但是，如果是结构体，我们可以给不同项存储不同数据类型
// 相当于 Java 的类，一个类有一到多个属性，每个属性之间可以有不同的数据类型
type Books struct {
	title   string
	author  string
	subject string
	book_id int
}

func main() {
	//println("创建一个结构体Books：", Books{"Go 语言", "www.xxx.com", "Go 教程", 1232})
	bookA := Books{"Go 语言", "www.xxx.com", "Go 教程", 1232}
	bookB := Books{title: "Python 语言", book_id: 4444}
	fmt.Printf("bookA %v\n", bookA) // bookA {Go 语言 www.xxx.com Go 教程 1232}
	fmt.Printf("bookB %v\n", bookB) // bookB {Python 语言   4444}

	println("结构体指针", &bookA)   // 0xc00006bf18
	println("结构体也能作为形参")
	printBook(bookA)
}

func printBook(book Books) {
	//println("printBook", book.title, book.author)
	fmt.Printf("%v\n", book)
}
```
# slice 实现Go动态数组
Go 语言切片是 对 数组 的抽象

因为：Go 数组 长度不可改变，在特定场景中，这样的不可变集合不适用。

于是：Go提供了 一种灵活、功能强悍的内置类型切片（“动态数组”）

声明格式： var id []type

```
func main() {
	//var badArr1 [3]int = [3]int{1, 2, 3}
	badArr := [...]int{1, 2, 3}
	//  动态数组初始化方式一
	//var goodArr1 []int  = make([]int, 3)
	//  动态数组初始化方式二
	//goodArr2 := make([]int, 3)
	//  动态数组初始化方式三
	goodArr := [] int{1, 2, 3}

	println("普通数组")
	for i := 0; i < len(badArr); i++ {
		print(badArr[i], ",")
	}
	println()
	println("可变数组")
	for i := 0; i < len(goodArr); i++ {
		print(goodArr[i], ",")
	}
	println()
	goodArr = append(goodArr, 4, 5, 6)
	println("可变数组加了几个元素后")
	printSlice(goodArr)

	println()
	println()
	println("开始切片")
	s := [] int{1, 2, 3}
	println("这里有个切片s(len = cap = 3)：")
	printSlice(s)
	println("获取切片s后面两个元素：")
	printSlice(s[1:])
	println()
	println("获取切片s后面两个元素初始化为另一个切片s1：")
	s1 := s[1:]
	printSlice(s1)

	println()
	println("-------------------------------------------------")
	println("len() 和 cap()")

	var nums = make([]int, 3,5)
	printSlice(nums)

	println()
	println("-------------------------------------------------")
	println("空切片（nil）")

	var numbers []int
	printSlice(numbers)
	if numbers == nil {
		fmt.Printf("切片是空的")
	}
	println()
	println("-------------------------------------------------")
	println("append() 和 copy() 函数")
	append_and_copy()
}

func append_and_copy() {
	var numbers []int
	printSlice(numbers)

	/* 允许追加空切片 */
	numbers = append(numbers, 0)
	printSlice(numbers)

	/* 向切片添加一个元素 */
	numbers = append(numbers, 1)
	printSlice(numbers)

	/* 同时添加多个元素 */
	numbers = append(numbers, 2,3,4)
	printSlice(numbers)

	/* 创建切片 numbers1 是之前切片的两倍容量*/
	numbers1 := make([]int, len(numbers), (cap(numbers))*2)

	/* 拷贝 numbers 的内容到 numbers1 */
	copy(numbers1,numbers)
	printSlice(numbers1)
}

func printSlice(x []int) {
	//for i := 0; i < len(goodArr); i++ {
	//	print(goodArr[i], ",")
	//}
	fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}
```
输出：
```
普通数组
1,2,3,
可变数组
1,2,3,
可变数组加了几个元素后
len=6 cap=6 slice=[1 2 3 4 5 6]


开始切片
这里有个切片s(len = cap = 3)：
len=3 cap=3 slice=[1 2 3]
获取切片s后面两个元素：
len=2 cap=2 slice=[2 3]

获取切片s后面两个元素初始化为另一个切片s1：
len=2 cap=2 slice=[2 3]

-------------------------------------------------
len() 和 cap()
len=3 cap=5 slice=[0 0 0]

-------------------------------------------------
空切片（nil）
len=0 cap=0 slice=[]
切片是空的
-------------------------------------------------
append() 和 copy() 函数
len=0 cap=0 slice=[]
len=1 cap=1 slice=[0]
len=2 cap=2 slice=[0 1]
len=5 cap=6 slice=[0 1 2 3 4]
len=5 cap=12 slice=[0 1 2 3 4]
```
# range
range 关键字：用于for 循环中 迭代数组array、切片slice、通道channel、集合map 的元素
* 数组和切片中，返回元素的索引和索引对应的值
* 集合中，返回key-value对
```
func main() {
	nums := []int{2, 3, 4}
	sum := 0
	// 当我们不想要这个index时，可以用"_"代替
	for _, num := range nums {
		sum += num
	}
	println("sum", sum) // 9
	for i, num := range nums {
		println("遍历过程中：索引", i, "值", num)
	}
	// 遍历过程中：索引 0 值 2
    // 遍历过程中：索引 1 值 3
    // 遍历过程中：索引 2 值 4


	// for map
	kvs := map[string]int{"A": 18, "B": 25}
	for k, v := range kvs {
		fmt.Printf("%s -> %d\n", k, v)
	}
	// A -> 18
    // B -> 25


	// for 枚举 Unicode字符串。第一个参数是字符的索引，第二个是字符（Unicode的值）本身。
	for i, c := range "go" {
		println(i, c)
	}
	// 0 103
    // 1 111

}
```
# map
集合
* 特点： 无序，默认为nil，内部使用hash表实现

初始化格式：
* 声明变量，默认 map 是 nil
  > `var map_variable map[key_data_type]value_data_type`
* 使用 make 函数
  > `map_variable := make(map[key_data_type]value_data_type)`

```
func main() {
	var countryCapitalMap map[string]string /*创建集合 */
	countryCapitalMap = make(map[string]string)

	/* map插入key - value对,各个国家对应的首都 */
	countryCapitalMap [ "France" ] = "巴黎"
	countryCapitalMap [ "Italy" ] = "罗马"
	countryCapitalMap [ "Japan" ] = "东京"
	countryCapitalMap [ "India " ] = "新德里"

	/*使用键输出地图值 */
	for country := range countryCapitalMap {
		fmt.Println(country, "首都是", countryCapitalMap [country])
	}

	/*查看元素在集合中是否存在 */
	capital, ok := countryCapitalMap [ "American" ] /*如果确定是真实的,则存在,否则不存在 */
	/*fmt.Println(capital) */
	/*fmt.Println(ok) */
	if (ok) {
		fmt.Println("American 的首都是", capital)
	} else {
		fmt.Println("American 的首都不存在")
	}

	println("-----------------------------------------")
	println("delete() 函数")
	println("-----------------------------------------")

	/* 创建map */
	countryCapitalMap2 := map[string]string{"France": "Paris", "Italy": "Rome", "Japan": "Tokyo", "India": "New delhi"}

	fmt.Println("原始地图")

	/* 打印地图 */
	for country := range countryCapitalMap2 {
		fmt.Println(country, "首都是", countryCapitalMap2 [ country ])
	}

	/*删除元素*/ delete(countryCapitalMap2, "France")
	fmt.Println("法国条目被删除")

	fmt.Println("删除元素后地图")

	/*打印地图*/
	for country := range countryCapitalMap2 {
		fmt.Println(country, "首都是", countryCapitalMap2 [ country ])
	}
}
```
# 接口
```
package main

// 定义接口
type Phone interface {
	call()
}

type NokiaPhone struct {
}
// 实现call() 方法
func (nokiaPhone NokiaPhone) call() {
	println("I am Nokia")
}

type IPhone struct {
}

// 实现call() 方法
func (iPhone IPhone) call() {
	println("I am iPhone")
}

func main() {
	var phone Phone

	phone = new(NokiaPhone)
	phone.call()

	phone = new(IPhone)
	phone.call()
}
```
# 异常处理（基础）
```
package main

import (
	"errors"
	"fmt"
	"math"
)

/**
Go 内置的 error类型 是一个接口类型，其定义如下：
type error interface {
  Error() string
}
*/

// 这是一个 利用了 error 接口的 方法
func mySqrt(f float64) (float64, error) {
	if f < 0 {
		return 0, errors.New("math: square root of negative number")
	}
	return math.Sqrt(f), nil
}

func main() {
	println("--------------------------")
	println("基础用法")
	println("--------------------------")
	result, err := mySqrt(-1)
	if err != nil {
		fmt.Printf("报错了：%v\n", err)
	} else {
		println("结果是：", result)
	}
}
```
输出：
```
--------------------------
基础用法
--------------------------
报错了：math: square root of negative number
```

# 异常处理（自定义 error 接口）
```
package main

import (
	"fmt"
)

// 自定义一个DivideError 结构 （除法运算错误）
type DivideError struct {
	dividee int
	divider int
}

// 实现 `error` 接口
func (de *DivideError) Error() string {
	strFormat := `
    Cannot proceed, the divider is zero.
    dividee: %d
    divider: 0
`
	return fmt.Sprintf(strFormat, de.dividee)
}

// 定义 `int` 类型除法运算的函数
func Divide(varDividee int, varDivider int) (result int, errorMsg string) {
	if varDivider == 0 {
		dData := DivideError{
			dividee: varDividee,
			divider: varDivider,
		}
		errorMsg = dData.Error()
		return
	} else {
		return varDividee / varDivider, ""
	}
}

func main() {
	// 正常情况
	if result, errorMsg := Divide(100, 10); errorMsg == "" {
		fmt.Println("100/10 = ", result)
	}
	// 当除数为零的时候会返回错误信息
	if _, errorMsg := Divide(100, 0); errorMsg != "" {
		fmt.Println("errorMsg is: ", errorMsg)
	}
}
```
输出：
```
100/10 =  10
errorMsg is:
    Cannot proceed, the divider is zero.
    dividee: 100
    divider: 0
```

# 异步
Go 语言支持并发，我们只需要通过 go 关键字来开启 goroutine 即可。

goroutine 是轻量级线程，goroutine 的调度是由 Golang 运行时进行管理的。

goroutine 语法格式： go 函数名( 参数列表 )

Go 允许使用 go 语句开启一个新的运行期线程， 即 goroutine，以一个不同的、新创建的 goroutine 来执行一个函数。
同一个程序中的所有 goroutine 共享同一个地址空间。

```
package main
import "time"

func say(s string) {
	for i := 0; i < 5; i++ {
		time.Sleep(100 * time.Millisecond)
		println(s)
	}
}

func main() {
	go say("world")
	say("hello")
	println("输出的 hello 和 world 是没有固定先后顺序。因为它们是两个 goroutine 在执行")
}
```
# channel 通道
通道（channel）
* 作用：用于传递数据。可用于 两个goroutine 之间通过传递一个指定类型的值来同步运行和通信。
* 操作符：`<-` 用于指定通道的方向、发送或接收。

如果未指定方向，则是 双向通道
```
	ch <- v    // 把 v 发送到通道 ch
	v := <-ch  // 从 ch 接收数据
			   // 并把值赋给 v
```
声明一个通道很简单，我们使用chan关键字即可，通道在使用前必须先创建：
```
  ch := make(chan int)
```
注意：默认情况下，通道是不带缓冲区的。发送端发送数据，同时必须有接收端相应的接收数据。
```
package main

import "fmt"

func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}
	c <- sum // 把 sum 发送到通道 c
}

func main() {
	println("--------------------------")
	println("例子：分段求和，最终相加")
	println("--------------------------")
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int)
	go sum(s[:len(s)/2], c)
	go sum(s[len(s)/2:], c)
	x, y := <-c, <-c // 从通道 c 中接收

	fmt.Println(x, y, x+y)

	println("--------------------------")
	println("通道缓冲区（通过 make 的第二个参数指定缓冲区大小）")
	println("--------------------------")
	/**
	带缓冲区的通道允许发送端的数据发送和接收端的数据获取处于异步状态，就是说发送端发送的数据可以放在缓冲区里面，可以等待接收端去获取数据，而不是立刻需要接收端去获取数据。

	不过由于缓冲区的大小是有限的，所以还是必须有接收端来接收数据的，否则缓冲区一满，数据发送端就无法再发送数据了。

	注意：如果通道不带缓冲，发送方会阻塞直到接收方从通道中接收了值。如果通道带缓冲，发送方则会阻塞直到发送的值被拷贝到缓冲区内；如果缓冲区已满，则意味着需要等待直到某个接收方获取到一个值。接收方在有值可以接收之前会一直阻塞。
	*/
	// 这里我们定义了一个可以存储整数类型的带缓冲通道
	// 缓冲区大小为2
	ch := make(chan int, 2)

	// 因为 ch 是带缓冲的通道，我们可以同时发送两个数据
	// 而不用立刻需要去同步读取数据
	ch <- 1
	ch <- 2

	// 获取这两个数据
	fmt.Println(<-ch)
	fmt.Println(<-ch)

	println("--------------------------")
	println("Go 遍历通道与关闭通道")
	println("--------------------------")
	/**
	Go 通过 range 关键字来实现遍历读取到的数据，类似于与数组或切片。格式如下：
	  v, ok := <-ch
	如果通道接收不到数据后 ok 就为 false，这时通道就可以使用 close() 函数来关闭。
	*/
	channelA := make(chan int, 10)
	go fibonacci(cap(channelA), channelA)
	// range 函数遍历每个从通道接收到的数据，因为 c 在发送完 10 个
	// 数据之后就关闭了通道，所以这里我们 range 函数在接收到 10 个数据
	// 之后就结束了。如果上面的 c 通道不关闭，那么 range 函数就不
	// 会结束，从而在接收第 11 个数据的时候就阻塞了。
	for i := range channelA {
		fmt.Println(i)
	}
}


func fibonacci(n int, c chan int) {
	x, y := 0, 1
	// 一边计算，一边扔到 通道的缓冲区中
	for i := 0; i < n; i++ {
		c <- x
		x, y = y, x+y
	}
	// 完了，通道没有生产者了，此时，就可以close通道了，close之后，通道不会再进数据，但是还是能够读取数据
	close(c)
}
```

# Go语言常用命令
## go get：一键获取代码、编译并安装
go get 命令可以借助代码管理工具通过远程拉取或更新代码包及其依赖包，并自动完成编译和安装。整个过程就像安装一个 App 一样简单。

这个命令可以动态获取远程代码包，目前支持的有 BitBucket、GitHub、Google Code 和 Launchpad。在使用 go get 命令前，需要安装与远程包匹配的代码管理工具，如 Git、SVN、HG 等，参数中需要提供一个包名。

这个命令在内部实际上分成了两步操作：
1. 第一步是下载源码包；
2. 第二步是执行 `go install`。

下载源码包的 go 工具会自动根据不同的域名调用不同的源码工具，对应关系如下：
```
BitBucket (Mercurial Git)
GitHub (Git)
Google Code Project Hosting (Git, Mercurial, Subversion)
Launchpad (Bazaar)
```
所以为了 go get 命令能正常工作，你必须确保安装了合适的源码管理工具，并同时把这些命令加入你的 PATH 中。其实 go get 支持自定义域名的功能。

参数：
```
-d 只下载不安装
-f 只有在你包含了 -u 参数的时候才有效，不让 -u 去验证 import 中的每一个都已经获取了，这对于本地 fork 的包特别有用
-fix 在获取源码之后先运行fix，然后再去做其他的事情
-t 同时也下载需要为运行测试所需要的包
-u 强制使用网络去更新包和它的依赖包
-v 显示执行的命令
-insecure 允许使用不安全的 HTTP 方式进行下载操作
```

一般用法：
```
go get -v -u <远程包：网站域名>/<作者>/<项目名>
```

# 项目github
https://github.com/androidjp/training-Go

# 相关学习链接
* https://studygolang.com/
* http://c.biancheng.net/view/123.html
* http://c.biancheng.net/golang/syntax/