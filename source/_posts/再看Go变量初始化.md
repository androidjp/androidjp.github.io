---
title: 再看Go变量初始化
date: 2020-10-12 16:12:41
tags:
- Go
categories:
- Go
---

> 我们通过几个小例子快速重温Golang变量初始化过程。
<!--more-->

## 看一个例子

让我们先看看这两个例子，对比一下：

case 1:

```Go
package main

import "fmt"

var (
    a int = b + 1
    b int = 1
)

func main() {
    fmt.Println(a)
    fmt.Println(b)
}
```

case 2：

```Go
package main

import "fmt"

func main() {
    var (
        a int = b + 1
        b int = 1
    )
    fmt.Println(a)
    fmt.Println(b)
}
```

哈哈， 两个case一对比，仅仅是变量从包内可见，变成局部变量，但输出结果却是不同：

```纯文本
case 1 -----正常
1
2

case 2 -----报错
xxx.go:7: undefined: b
```



**原因**：不同作用域类型的变量初始化顺序不同。

- case 2中的a,b 是局部变量（作用域：方法），初始化顺序：从左到右、从上到下；

- case 1中package级别的变量，初始化顺序与**初始化依赖**有关；



看看这个例子：

```Go
package main

import "fmt"

var (
  a = c - 2
  b = 2
  c = f()
)

func f() int {
  fmt.Printf("inside f and b = %d\n", b)
  return b + 1
}
func main() {
  fmt.Println(a)
  fmt.Println(b)
  fmt.Println(c)
}

```

输出：

```纯文本
inside f and b = 2
1
2
3
```

说明：

a依赖c，c依赖b，于是，初始化顺序：

1. b = 2

2. c = f() = 3

3. a = c - 2 = 1



## 变量循环依赖

基于这个变量依赖的规则，就可能存在这个变量循环依赖的问题：

```Go
package main
import "fmt"

var (
    a = b
    b = c
    c = f()
)
func f() int {    
    return a
}
func main() {
    fmt.Println(a, b, c)
}
```

输出：

```纯文本
错误：“initialization loop”
```

