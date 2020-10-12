---
title: Go错误与异常处理实践总结
date: 2020-10-12 16:22:41
tags:
- Go
categories:
- Go
---

<!--more-->

# 我们认识的error

`error`看着是一个关键字，但本身就是一个接口，位于`buildin.go`中，

```Go
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
  Error() string
}
```

而`errors.New(string)`位于`errors/errors.go`中，背后返回的是error的最简单实现：

```Go
// errors.New() 方法
func New(text string) error {
  return &errorString{text}
}

// 最简单实现：里头就一个错误信息字符串
type errorString struct {
  s string
}

// 返回错误信息字符串
func (e *errorString) Error() string {
  return e.s
} 
```

# 一般的场景用法

平常我们写方法的异常处理，一般是这样子：

```Go
func Method01(url string) (interface{}, error) {
    if err:= ConnectDB(url); err != nil {
        return nil, err
    }
    // .....
    return "xxx", nil
}  
```

或者这样：

```Go
func Method01(url string) (interface{}, error) {
    if err:= ConnectDB(url); err != nil {
        return nil, errors.New("Connect DB failed!!")
    }
    // .....
    return "xxx", nil
}  
```

好一点就把相关的检索用的关键参数一并记录：

```Go
func Method01(url string) (interface{}, error) {
    if err:= ConnectDB(url); err != nil {
        return nil, fmt.Errorf("Connect DB failed!! url=[%s], %v", url, err.Error())
    }
    // .....
    return "xxx", nil
}
```

我们不妨多理解一下Go中的错误和异常 两者的异同，在实现业务逻辑时，才不会疑惑为何方法要这么写异常处理逻辑，要怎么返回error对象。

# 再次理解错误与异常

我们抛开语言特性，如Java中的Error和Exception的角度，而用更为直白的方式去看错误和异常：

- 错误【意料之中】：可能出现问题的地方出现了问题，如打开文件失败，连接数据库失败等。

- 异常【意料之外】：不应该出现问题的地方出现了问题，如引用了空指针。

**错误是业务过程的一部分，而异常不是。**



## Go世界里的错误

回到Golang的世界，Golang中的`error`以及以上的几个例子，都是在**错误**这一范畴。Go想要引入error接口作为错误处理的标准模式，如果函数要返回错误，则返回值类型列表中肯定包含error。error的处理过程类似C语言中的错误码，可逐层返回，直到被处理。

## Go世界里的异常

Golang引入两个内置函数：

- `panic`：触发异常处理流程

- `recover`：终止异常处理流程

同时引入关键字`defer`，来延迟执行`defer`后面的函数，一直等到包含`defer`语句所在的函数执行完毕，延迟函数（`defer`后面跟着的函数）才会被执行，而不管包含`defer`语句的函数时通过return语句正常结束，还是由于`panic`导致的异常结束。

当发生以下等等情况时：

- 引用空指针

- 下标越界

- 显示调用`panic`

- ....

，会先触发`panic`函数，然后调用延迟函数。调用者继续传递`panic`，因此该过程一直在调用栈中重复发生：函数停止执行，调用延迟执行函数等。如果一路在延迟函数中没有`recover`函数的调用，则会到达该协程的起点，该协程结束，然后终止其他所有协程，包括主协程（类似于C语言中的主线程，该协程ID为1）。



## 错误与异常，之于Go

错误和异常从Golang机制上讲，就是error和panic的区别。

很多其他语言也一样，比如C++/Java，没有error但有errno，没有panic但有throw。



## Go错误和异常可以相互转换

- 错误转异常：

- 比如程序逻辑上尝试请求某个URL，最多尝试三次，前两次请求失败时错误，第三次失败，错误就会被提升为异常；

- 异常转错误：

- 比如`panic`触发的异常被`recover`恢复后，将返回值中error类型的变量进行赋值，以便上层函数继续走错误处理流程；



# 看源码，得启发

重新认识完Golang世界里的错误和异常，让我们一起看个身边源码例子：

`regexp.go`中的两个函数：

```Go
// 编译
func Compile(expr string) (*Regexp, error) {
  return compile(expr, syntax.Perl, false)
}

// 必须编译
func MustCompile(str string) *Regexp {
  regexp, err := Compile(str)
  if err != nil {
    panic(`regexp: Compile(` + quote(str) + `): ` + err.Error())
  }
  return regexp
} 
```

虽然背后都是调用compile()函数，做的都是“编译”，但设计不同：

- Compile函数**基于错误处理设计**，将正则表达式编译成有效的可匹配格式，适用于用户输入场景。当用户输入的正则表达式不合法时，该函数会返回一个错误。

- MustCompile函数**基于异常处理设计**，适用于硬编码场景。当调用者明确知道输入不会引起函数错误时，要求调用者检查这个错误是不必要和累赘的。我们应该假设函数的输入一直合法，当调用者输入了不应该出现的输入时，就触发panic异常。

于是，我们得到了一个启示：**什么情况下用错误表达，什么情况下用异常表达，就得有一套规则，否则很容易出现一切皆错误或一切皆异常的情况**。



## 启发：适用于异常处理的场景

想一想，我们是否可以归纳出常见得一些场景，这些场景，是需要做异常处理得：

1. 空指针

2. 下标越界

3. 除数为0

4. 不应该出现得分支，如：default

5. 输入不应该引起函数错误



那么，其他场景，就使用错误处理。

这样，就使得我们在设计接口时更加清晰。

另外，对于异常，我们可以选择在一个合适的上游去recover，并打印堆栈信息，使得部署后的程序不会终止。



# 实践起来，让代码变漂亮

看到上面的例子，就知道，我们程序代码中一般处理错误的方式，就是`if err != nil {...}` 这种方式对实际业务逻辑代码的入侵是很大的，让人看着难受~！



## 错误处理的正确姿势

### 一、失败原因只有一个时，不使用error

```Go
func (self *AgentContext) CheckHostType(host_type string) error {
    switch host_type {
    case "virtual_machine":
        return nil
    case "bare_metal":
        return nil
    }
    return errors.New("CheckHostType ERROR:" + host_type)
}
```

上述函数的作用其实就是：判断输入是否是某几个值的其中一个，不是就报错。

分析一下：

- 方法没有意料之外的事情会发送，最多是属于错误处理场景；

- 报错只有一种情况，错误没有再分不同情况；

那我用一个bool值不更简单更好被理解？于是变成这样：

```Go
func (self *AgentContext) CheckHostType(host_type string) bool {
    return host_type == "virtual_machine" || host_type == "bare_metal"
}
```



### 二、没有失败时，不使用error

例子：

```Go
func (c *Client) setTargetId() error {
    self.TargetId = self.Id
    return nil
}

// .....

// 调用方
func (c *Client) TryConnect() error { 
    // .........
    err := c.setTargetId()
    if err != nil {
        // log
        // free resource
        return errors.New(....)
    }
}
```

分析：

- 不存在报错问题，方法setTargetId不需要返回error；

- 由于setTargetId返回的error，调用方需要做error的处理；

重构一下，得到：

```Go
func (c *Client) setTargetId() {
    self.TargetId = self.Id
}

// .....

// 调用方
func (c *Client) TryConnect() error { 
    // .........
    c.setTargetId()
    // .........
}
```

### 三、error应放在返回值类型列表的最后

这个应该是一大基本规范了：

```Go
resp, err := http.Get(url)
if err != nil {
    return nill, err
}
```

当然，bool作为返回值类型时，也一样，放在返回列表最后：

```Go
value, ok := cache.Lookup(key) 
if !ok {
    // ...cache[key] does not exist… 
}
```

### 四、错误值统一定义，而不是跟着感觉走

项目中，到处都是 return errors.New(value)，已经让人有些烦躁了，错误内容还要花样多多且意思雷同，过分了！：

1. "record is not existed."

2. "record is not exist!"

3. "###record is not existed！！！"

4. ...



不良影响：

1. 调用方无法统一标准来handle这些error；

2. 万一又新增了一个奇奇怪怪的error，调用方handle不了，又得上代码；



所以，我们可以定义错误码文件，将错误对象定义在一个相对公共的文件中：

```Go
var ERR_EOF = errors.New("EOF")
var ERR_CLOSED_PIPE = errors.New("io: read/write on closed pipe")
var ERR_NO_PROGRESS = errors.New("multiple Read calls return no data or error")
var ERR_SHORT_BUFFER = errors.New("short buffer")
var ERR_SHORT_WRITE = errors.New("short write")
var ERR_UNEXPECTED_EOF = errors.New("unexpected EOF")
```

当然，命名方式可以跟随团队规范。



### 五、错误逐层传递时，层层都加日志

目的：方便故障定位



### 六、错误处理使用defer

例子：我想要创建4个资源，任何一个资源创建失败，就得回滚并销毁先前创建的资源。

```Go
func deferDemo() error {
    err := createResource1()
    if err != nil {
        return ERR_CREATE_RESOURCE1_FAILED
    }
    err = createResource2()
    if err != nil {
        destroyResource1()
        return ERR_CREATE_RESOURCE2_FAILED
    }

    err = createResource3()
    if err != nil {
        destroyResource1()
        destroyResource2()
        return ERR_CREATE_RESOURCE3_FAILED
    }

    err = createResource4()
    if err != nil {
        destroyResource1()
        destroyResource2()
        destroyResource3()
        return ERR_CREATE_RESOURCE4_FAILED
    }
    return nil
}
```

分析：

- 上述方法的异常处理很容易写错；

- 完全可以利用defer“先进后出”的特性，配合闭包，就做到回滚的效果；

- 对闭包来说，参数是值传递，对于外部变量却是引用传递，所以闭包中的外部变量err的值就变成外部函数返回时最新的err值；

于是，重构后，得到：

```Go
func deferDemo() error {
    err := createResource1()
    if err != nil {
        return ERR_CREATE_RESOURCE1_FAILED
    }
    defer func() {
        if err != nil {
            destroyResource1()
        }
    }()
    err = createResource2()
    if err != nil {
        return ERR_CREATE_RESOURCE2_FAILED
    }
    defer func() {
        if err != nil {
            destroyResource2()
        }
    }()

    err = createResource3()
    if err != nil {
        return ERR_CREATE_RESOURCE3_FAILED
    }
    defer func() {
        if err != nil {
            destroyResource3()
        }
    }()

    err = createResource4()
    if err != nil {
        return ERR_CREATE_RESOURCE4_FAILED
    }
    return nil
}
```

### 七、尝试几次可以避免失败时，不要立即返回错误

像请求URL、连接或访问DB等场景，都适用。



### 八、当上层函数不关心错误时，建议不返回error

对于一些资源清理相关的函数（destroy/delete/clear），如果子函数出错，打印日志即可，而无需将错误进一步反馈到上层函数，因为一般情况下，上层函数是不关心执行结果的，或者即使关心也无能为力，于是我们建议将相关函数设计为不返回error。



### 九、当发生错误时，不忽略有用的返回值

像 `func Exec() (int64, error)` 这样的DB命令执行函数，返回值是：

- 修改成功数量：int64

- 执行报错：error

这样一来，我们就得关注一下返回值中的这些有用参数，而非只关注error。



## 异常处理的正确姿势

### 一、程序开发阶段，坚持速错

### 二、程序部署后，应恢复异常避免程序终止

要想想，如果不恢复，那么最终异常会终止包括主协程在内的所有协程，最终终止整个进程。

所以，一旦Golang程序部署后，在任何情况下发生的异常都不应该导致程序异常退出，我们在上层函数中加一个延迟执行的recover调用来达到这个目的，并且是否进行recover需要根据环境变量或配置文件来定，默认需要recover。

我们在调用recover的延迟函数中以最合理的方式响应该异常：

1. 打印堆栈的异常调用信息和关键的业务信息，以便这些问题保留可见；

2. 将异常转换为错误，以便调用者让程序恢复到健康状态并继续安全运行。

看个简单例子：

```Go
func funcA() error {
    defer func() {
        if p := recover(); p != nil {
            fmt.Printf("panic recover! p: %v", p)
            debug.PrintStack()
        }
    }()
    return funcB()
}

func funcB() error {
    // simulation
    panic("foo")
    return errors.New("success")
}

func test() {
    err := funcA()
    if err == nil {
        fmt.Printf("err is nil\\n")
    } else {
        fmt.Printf("err is %v\\n", err)
    }
}
```

我们期望输出：`err is foo`

但是实际输出：`err is nil`

原因：panic异常处理机制不会自动将错误信息传递给error，所以要在funcA函数中进行显式地传递：

```Go
func funcA() (err error) {
    defer func() {
        if p := recover(); p != nil {
            fmt.Println("panic recover! p:", p)
            str, ok := p.(string)
            if ok {
                err = errors.New(str)
            } else {
                err = errors.New("panic")
            }
            debug.PrintStack()
        }
    }()
    return funcB()
}
```

### 三、对于不应该出现的分支，使用异常处理

如，程序到达了某条逻辑上不可能到达的路径：

```Go
switch s := suit(drawCard()); s {
    case "Spades":
    // ...
    case "Hearts":
    // ...
    case "Diamonds":
    // ... 
    case "Clubs":
    // ...
    default:
        panic(fmt.Sprintf("invalid suit %v", s))
}
```

### 四、针对入参不应该有问题的函数，使用异常

入参不应该有问题一般指的是硬编码，而非用户输入。（换句话说，程序员自己写下的配置或者代码，不应该有错）


# 总结

1. 在Golang看来，错误是业务过程的一部分，而异常不是；
2. Golang中，错误处理看`error`，异常处理看`panic`、`recover`；
3. 划分好错误处理和异常处理的使用场景，很关键；


# 参考资料

[https://www.jianshu.com/p/f30da01eea97](https://www.jianshu.com/p/f30da01eea97)