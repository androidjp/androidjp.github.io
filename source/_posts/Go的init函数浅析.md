---
title: Go的init函数浅析
date: 2020-10-12 16:02:41
tags:
- Go
categories:
- Go
---


> go中除了 返回即终止的main函数之外，还有一个init函数。

<!--more-->

## 主要作用

- 初始化不能采用初始化表达式初始化的变量；

```Go
var initArg [20]int
func init() {
   initArg[0] = 10
   for i := 1; i < len(initArg); i++ {
       initArg[i] = initArg[i-1] * 2
   }
}
```

- 程序运行前的注册；

- 实现sync.Once功能；

## 特点

- 运行程序时，先于main函数执行；

- 不能被其他函数调用；

- 没有入参、没有返回值；

- 每个包可以用N个init函数；

- 每个.go文件中可以用N个init函数；

- 同一个包的init执行顺序，golang没有明确定义，编程时要注意程序不要依赖这个执行顺序。

- 不同包的init函数按照包导入的依赖关系决定执行顺序。



## golang程序初始化过程

golang程序初始化先于main函数执行，由runtime进行初始化，初始化顺序如下：

1. 初始化导入的包（包的初始化顺序并不是按导入顺序（“从上到下”）执行的，runtime需要解析包依赖关系，没有依赖的包最先初始化，与变量初始化依赖关系类似）；

2. 初始化包作用域的变量（该作用域的变量的初始化也并非按照“从上到下、从左到右”的顺序，runtime解析变量依赖关系，没有依赖的变量最先初始化）；

3. 执行包的init函数；



### 例子：变量初始化>init()>main()

```Go
package main                                                                                                                     

import (
   "fmt"              
)

var T int64 = a()

func init() {
   fmt.Println("init in main.go")
}

func a() int64 {
   fmt.Println("calling a()")
   return 2
}
func main() {                  
   fmt.Println("calling main")     
}
```

输出：

```纯文本
calling a()
init in main.go
calling main 
```

初始化顺序：**变量初始化**->**init()**->**main()**



### 例子：多个包之间，被依赖的先初始化

pack/pack.go

```Go
package pack
import (
  "demo01/util"
  "fmt"
)

var Pack int = 6

func init() {
  a := util.Util
  fmt.Println("init pack ", a)
}
```

util/util.go

```Go
package util

import "fmt"

var Util int = a()
func a() int {
  fmt.Println("util a()")
  return 5
}

func init() {
  fmt.Println("init util")
}

func init() {
 fmt.Println("init util time2")
}

func init() {
 fmt.Println("init util time3")
}  
```

util/util2.go

```Go
package util

import "fmt"

func init() {
  fmt.Println("init util2")
}
```

util/autil.go

```Go
package util

import "fmt"

func init() {
  fmt.Println("init autil")
}

var B int = b()

func b() int {
  fmt.Println("autil b()")
  return 11
} 
```

main.go

```Go
package main

import (
  "demo01/pack"
  "demo01/util"
  "fmt"
)

func main() {
  fmt.Println("main()", pack.Pack)
  fmt.Println("main()", util.Util)
}

```

最终输出：

```纯文本
autil b()
util a()
init autil
init util
init util time2
init util time3
init util2
init pack  5
main() 6
main() 5

```

初始化顺序：

1. 被依赖的包的所有变量初始化操作，按同包中源文件命名字典序依次执行；

2. 被依赖的包的所有init函数，按同包中源文件命名字典序依次执行；

3. 依赖别包的当前包的init函数；



### 例子：只想跑init，不想导出变量或方法

```Go
import _ "net/http/pprof"
```

**golang对没有使用的导入包会编译报错，但是有时我们只想调用该包的init函数，不使用包导出的变量或者方法，这时就采用上面的导入方案。**

执行上述导入后，init函数会启动一个异步协程采集该进程实例的资源占用情况，并以http服务接口方式提供给用户查询。

