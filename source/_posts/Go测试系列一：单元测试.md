---
title: Go测试系列一：单元测试
date: 2021-03-23 14:02:41
tags:
- Go
- 单元测试
categories:
- Go
---

> 本篇文章将带大家入门Golang单元测试，笔者参考不同资料及手动demo，和大家一同看看Golang世界怎么写测试，为Golang TDD做热身。

<!--more-->

# 常用库

```Go
testing // 系统自带testing库（必备）
```

```Go
go get github.com/stretchr/testify // 断言库
```
`testify`库让你写出可读性更好的测试断言：

![](/images/20210323/1.png)


# 简单例子

Go 语言推荐测试文件和源代码文件放在一块，测试文件以 _test.go 结尾。比如，当前 package 有 calc.go 一个文件，我们想测试 calc.go 中的 Add 和 Mul 函数，那么应该新建 calc_test.go 作为测试文件。

```JavaScript
example/
|--calc.go  
|--calc_test.go 
```

calc.go

```Go
package main

func Add(a int, b int) int {
  return a + b
}

func Mul(a int, b int) int {
  return a * b
} 
```

calc_test.go

```Go
package main

import "testing"

/**
cd到unittest目录，然后直接命令行运行： go test
如果想要显示详细的每个测试方法是否验证成功： go test -v
如果想指定只跑某个测试：go test -run TestAdd -v

t.Fatal/t.Fatalf 遇错即停
t.Error/t.Errorf 遇错不停
*/
func TestAdd(t *testing.T) {
  if ans := Add(1, 2); ans != 3 {
    t.Errorf("1 + 2 expected be 3, but %d got", ans)
  }

  if ans := Add(-10, -20); ans != -30 {
    t.Errorf("-10 + -20 expected be -30, but %d got", ans)
  }
}

func TestMul(t *testing.T) {
  if ans := Mul(1, 2); ans != 2 {
    t.Errorf("1 + 2 expected be 2, but %d got", ans)
  }

  if ans := Mul(-10, -20); ans != 200 {
    t.Errorf("-10 + -20 expected be 200, but %d got", ans)
  }
}
```

## 如何运行测试？

cd到unittest目录，然后直接命令行运行： `go test`
如果想要显示详细的每个测试方法是否验证成功： `go test -v`
如果想指定只跑某个测试：`go test -run TestAdd -v`


另外：

- `t.Fatal/t.Fatalf` 遇错即停

- `t.Error/t.Errorf` 遇错不停

详细命令看这里：
```
go test  // 查找当前目录下的用例文件
go test pkg // pkg包里的所有示例
go test helloworld_test.go //指定用例文件
go test -v -run TestA select_test.go //指定文件的单个单元用例运行。'-v' 打印详细信息
go test -v -bench=. benchmark_test.go // 指定文件的某个性能用例运行。'.' 表示所有性能用例
go test -v -bench=. -benchtime=5s benchmark_test.go // '-benchtime=5s' 指定测试时长，默认1s
go test -v -bench=Alloc -benchmem benchmark_test.go // 指定单个性能用例
go test -cover  //覆盖率
go test ./... -v -cover // 运行当前目录以及子目录的所有的测试文件，并展示详细信息、展示覆盖率 
```

# subtest 子测试

cal_test.go

```Go
.....
func TestMul_SubTests(t *testing.T) {
  t.Run("should_return_6_when_Mul_given_2_and_3", func(t * testing.T) {
    if ans := Mul(2, 3); ans != 6 {
      t.Fatal("fail")
    }
  })

  t.Run("should_return_negative_6_when_Mul_given_2_and_negative_3", func(t * testing.T) {
    if ans := Mul(2, -3); ans != -6 {
      t.Fatal("fail")
    }
  })
} 
```

关键语句：

```Go
t.Run
```



# table-driven tests 测试

我们更加推崇这种写法：

demo2.go

```Go
package main

func MergeString(x, y string) string {
  return x + y
}
```

demo2_test.go

```Go
package main

import "testing"

func TestMergeString(t *testing.T) {
  tests := []struct {
    name string
    X, Y, Expected string
  }{
    {"should_return_HelloWorld_when_MergeString_given_Hello_and_World", "Hello", "World", "HelloWorld"},
    {"should_return_aaaBBB_when_MergeString_given_aaa_and_BBB", "aaa", "bbb", "aaaBBB"},
  }
  for _, test := range tests {
    t.Run(test.name, func(t *testing.T) {
      if ans:= MergeString(test.X, test.Y);ans!=test.Expected {
        t.Error("fail")
      }
    })
  }
}
```

所有用例的数据组织在切片 `cases` 中，看起来就像一张表，借助循环创建子测试。这样写的**好处**有：

- 新增用例非常简单，只需给 cases 新增一条测试数据即可。

- 测试代码可读性好，直观地能够看到每个子测试的参数和期待的返回值。

- 用例失败时，报错信息的格式比较统一，测试报告易于阅读。

如果数据量较大，或是一些二进制数据，推荐使用相对路径从文件中读取



# 帮助函数Helper

demo3_test.go

```Go
package main

import "testing"

type myCase struct {
  Str string
  Expected string
}

// 帮助函数：用于将某些 common 的 code，重构出来
func createFirstLetterToUpperCase(t *testing.T, c *myCase) {
  // to.Helper() 用于在 运行  go test 时能够打印出报错对应的行号
  t.Helper()
  if ans := FirstLetterToUpperCase(c.Str); ans != c.Expected {
    t.Errorf("input is `%s`, expect output is `%s`, but actually output `%s`", c.Str, c.Expected, ans)
  }
}

func TestFirstLetterToUpperCase(t *testing.T) {
  createFirstLetterToUpperCase(t, &myCase{"hello", "Hello"})
  createFirstLetterToUpperCase(t, &myCase{"ok", "Ok"})
  createFirstLetterToUpperCase(t, &myCase{"Good", "Good"})
  createFirstLetterToUpperCase(t, &myCase{"GOOD", "Good"})
}
```

demo3.go

```Go
package main

import "strings"

func FirstLetterToUpperCase(x string) string {
  return strings.ToUpper(x[:1]) + x[1:]
}

```

关键语法：

```Go
to.Helper() // 用于在 运行  go test 时能够打印出报错对应的行号
```

关于 `helper` 函数的 2 个建议：

- 不要返回错误， 帮助函数内部直接使用 `t.Error` 或 `t.Fatal` 即可，在用例主逻辑中不会因为太多的错误处理代码，影响可读性。

- 调用 `t.Helper()` 让报错信息更准确，有助于定位。

# setup 和 teardown

其实就是 一些test 的生命周期相关函数

setup可以做一些初始化操作，teardown可以做一些资源回收工作。

```Go
func setup() {
  fmt.Println("Before all tests")
}

func teardown() {
  fmt.Println("After all tests")
}

func Test1(t *testing.T) {
  fmt.Println("I'm test1")
}

func Test2(t *testing.T) {
  fmt.Println("I'm test2")
}

func TestMain(m *testing.M) {
  setup()
  code := m.Run()
  teardown()
  os.Exit(code)
}
```

说明：

- 在这个测试文件中，包含有2个测试用例，`Test1` 和 `Test2`。

- 如果测试文件中包含函数 `TestMain`，那么生成的测试将调用 TestMain(m)，而不是直接运行测试。

- 调用 `m.Run()` 触发所有测试用例的执行，并使用 `os.Exit()` 处理返回的状态码，如果不为0，说明有用例失败。

- 因此可以在调用 `m.Run()` 前后做一些额外的准备(setup)和回收(teardown)工作。

执行 `go test`，将会输出：

```Go
$ go test
Before all tests
I'm test1
I'm test2
PASS
After all tests
ok      example 0.006s

```

# 网络测试(Network)

## TCP/HTTP

假设需要测试某个 API 接口的 handler 能够正常工作，例如 helloHandler

```Go
func helloHandler(w http.ResponseWriter, r *http.Request) {
  w.Write([]byte("hello world"))
}
```

那我们可以创建真实的网络连接进行测试：

```Go
// test code
import (
  "io/ioutil"
  "net"
  "net/http"
  "testing"
)

func handleError(t *testing.T, err error) {
  t.Helper()
  if err != nil {
    t.Fatal("failed", err)
  }
}

func TestConn(t *testing.T) {
  ln, err := net.Listen("tcp", "127.0.0.1:0")
  handleError(t, err)
  defer ln.Close()

  http.HandleFunc("/hello", helloHandler)
  go http.Serve(ln, nil)

  resp, err := http.Get("http://" + ln.Addr().String() + "/hello")
  handleError(t, err)

  defer resp.Body.Close()
  body, err := ioutil.ReadAll(resp.Body)
  handleError(t, err)

  if string(body) != "hello world" {
    t.Fatal("expected hello world, but got", string(body))
  }
}
```

- `net.Listen("tcp", "127.0.0.1:0")`：监听一个未被占用的端口，并返回 Listener。

- 调用 `http.Serve(ln, nil)` 启动 http 服务。

- 使用 `http.Get` 发起一个 Get 请求，检查返回值是否正确。

- 尽量不对 `http` 和 `net` 库使用 mock，这样可以覆盖较为真实的场景。

## httptest

针对 http 开发的场景，使用标准库 net/http/httptest 进行测试更为高效。

上述的测试用例改写如下：

```Go
// test code
import (
  "io/ioutil"
  "net/http"
  "net/http/httptest"
  "testing"
)

func TestConn(t *testing.T) {
  req := httptest.NewRequest("GET", "http://example.com/foo", nil)
  w := httptest.NewRecorder()
  helloHandler(w, req)
  bytes, _ := ioutil.ReadAll(w.Result().Body)

  if string(bytes) != "hello world" {
    t.Fatal("expected hello world, but got", string(bytes))
  }
}
```

使用 httptest 模拟请求对象(req)和响应对象(w)，达到了相同的目的。

# Benchmark 基准测试

基准测试用例的定义如下：

```Go
func BenchmarkName(b *testing.B){
    // ...  
}
```

- 函数名必须以 `Benchmark` 开头，后面一般跟待测试的函数名

- 参数为 `b *testing.B`。

- 执行基准测试时，需要添加 `-bench` 参数。

例如：

```Go
func BenchmarkHello(b *testing.B) {  
    for i := 0; i < b.N; i++ {
        fmt.Sprintf("hello")
    }
}
```

运行：`go test -benchmem -bench .`
得到结果：

```Go
BenchmarkHello-16   15991854   71.6 ns/op   5 B/op   1 allocs/op    
```

基准测试报告每一列值对应的含义如下：

```Go
type BenchmarkResult struct {  
    N         int           // 迭代次数  
    T         time.Duration // 基准测试花费的时间  
    Bytes     int64         // 一次迭代处理的字节数  
    MemAllocs uint64        // 总的分配内存的次数  
    MemBytes  uint64        // 总的分配内存的字节数  
}
```

如果在运行前基准测试需要一些耗时的配置，则可以使用 `b.ResetTimer()` 先重置定时器，例如：

```Go
func BenchmarkHello(b *testing.B) {
    ... // 耗时操作
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        fmt.Sprintf("hello")
    }
}
```

使用 `RunParallel` 测试并发性能

```Go
func BenchmarkParallel(b *testing.B) {
    templ := template.Must(template.New("test").Parse("Hello, {{.}}!"))
    b.RunParallel(func(pb *testing.PB) {
    var buf bytes.Buffer
    for pb.Next() {
    // 所有 goroutine 一起，循环一共执行 b.N 次
    buf.Reset()
    templ.Execute(&buf, "World")
    }
    })
}
```

```Go
$ go test -benchmem -bench .
...
BenchmarkParallel-16   3325430     375 ns/op   272 B/op   8 allocs/op
...
```