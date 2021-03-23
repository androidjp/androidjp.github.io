---
title: Go测试系列二：Mock实践
date: 2021-03-23 15:02:41
tags:
- Go
- 单元测试
categories:
- Go
---

> 上一篇文章将带大家入门Golang单元测试，接下来，就是怎么Mock。`Mock`也就是‘模拟’，也就是模拟一些上下文环境，来制造你想要的某个特定条件，来看看你的待测逻辑，是否在此特定条件下，正确运行了你认为的逻辑。

<!--more-->

以下就不说太多Golang和Java等其他语言的Mock测试环境的对比等等，我们直接上干货（附上Demo，每个分支都有对应的Mock案例）。

Demo Github 地址：https://github.com/androidjp/go-mock-best-practice



# gomock + mockgen

## 需求

- 接口打桩。对某个可赋值的依赖成员对象进行mock（比如 `ServiceA` 依赖 `RepositoryA`，那么，在测试 `ServiceA.MethodA`方法时，可以mock了`RepositoryA`）

## 安装与配置

1. 拉取安装gomock 和 mockgen
    ```Bash
    go get -v -u github.com/golang/mock/gomock
    ```
    得到 $GOPATH/src/github.com/golang/mock 目录下，有GoMock包和mockgen工具 两个子目录。
    第2，3，4步看你的$GOPATH/bin目录有没有已经安装好的mockgen可执行文件，有则可忽略后续步骤。

2. 进入mockgen子目录，执行build命令，即生成了可执行程序mockgen；
3. 将mockgen拷贝到$GOPATH/bin目录下；
4. 指定环境变量Path包含$GOPATH/bin目录；
5. 最后，尝试敲一下命令行：
    ```Go
    mockgen help
    ```
    如果出现`-bash: mockgen: command not found`，表示你的环境变量PATH中没有配置`$GOPATH/bin`。

## 文档

```纯文本
go doc github.com/golang/mock/gomock
```

[在线参考文档](https://link.jianshu.com/?t=http://godoc.org/github.com/golang/mock/gomock)

## mockgen使用

1. 在项目根目录打开命令行
2. 找到对应目录下的某个将你要mock的接口所在的.go文件，生成对应的mock文件
    ```纯文本
    mockgen -source=1_gomock/db/repository.go  > test/1_gomock/db/mock_repository.go
    ```
    当然，前提是你这个 `test/1_gomock/db/`目录已经存在。

3. 然后，使用这个mock文件中的 `MockXxx(t)` 方法

## 关键用法

### 1. 接口打桩步骤

1. 首先，使用mockgen工具，将对应的接口生成mock文件
2. 然后，开始打桩
    ```Go
    // 1. 初始化 mock控制器
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    // 2. 初始化mock对象，并注入控制器
    mockRepo := mock_gomock_db.NewMockRepository(ctrl)

    // 3. 设定mock对象的返回值
    mockRepo.EXPECT().Create("name", []byte("jasper")).Return(nil)
    ```

3. 然后，测你要测的逻辑
    ```Go
    // when
    demoSvr := &gomock_service.DemoService{Repo: mockRepo}
    data, err := demoSvr.InsertData("name", "jasper")
    // then
    assert.Equal(t, "success", data)
    assert.Nil(t, err)
    ```

### 2. 接口打桩定义前N次返回值

```Go
// 前两次返回错误
mockRepo.EXPECT().Create("name", []byte("jasper")).Return(errors.New("db connection error")).Times(2)
// 第三次正常
mockRepo.EXPECT().Create("name", []byte("jasper")).Return(nil)
```

### 3. 断言接口调用顺序

方式一：`After`

```Go
// retrieve 先执行
retrieveName := mockRepo.EXPECT().Retrieve("name").Return([]byte("jasper"), nil)
// update 在 retrieve 之后
mockRepo.EXPECT().Update("name", []byte("mike")).Return(nil).After(retrieveName) 
```

方式二：`InOrder`

```Go
gomock.InOrder(
    // retrieve 先执行
    mockRepo.EXPECT().Retrieve("name").Return([]byte("jasper"), nil),
    // update 在 retrieve 之后
    mockRepo.EXPECT().Update("name", []byte("mike")).Return(nil),
)
```

## Demo示例

[https://github.com/androidjp/go-mock-best-practice/tree/1_gomock_1_basic](https://github.com/androidjp/go-mock-best-practice/tree/1_gomock_1_basic)

# gostub打桩

## 需求

- 全局变量打桩

- 函数打桩

- 过程打桩

- 第三方库打桩

## 安装与配置

```Bash
go get -v -u github.com/prashantv/gostub
```

## 关键用法

### 1. 全局变量打桩

对于全局变量：

```Go
var (
  GlobalCount int
  Host        string
)
```

可以这样打桩：

```Go
// 全局变量 GlobalCount int 打桩
// 全局变量 Host string 打桩
stubs := gostub.Stub(&GlobalCount, 10).
  Stub(&Host, "https://www.bing.cn")
defer stubs.Reset() 
```

### 2. 函数打桩

假设有个函数：

```Go
func Exec(cmd string, args ...string) (string, error) {
  return "", nil
}
```

那么，首先我要先变成这样的写法：

```Go
var Exec = func(cmd string, args ...string) (string, error) {
  return "", nil
}
```

以上写法不影响业务逻辑使用。

然后再进行打桩：

方式一：`StubFunc` 直接设置返回结果

```Go
stubs := gostub.StubFunc(&gomock_service.Exec, "xxx-vethName100-yyy", nil)
defer stubs.Reset()
```

方式二：`Stub` 还能设置具体逻辑

```Go
stubs := gostub.Stub(&Exec, func(cmd string, args ...string) (string, error) {
      return "xxx-vethName100-yyy", nil
    })
defer stubs.Reset()
```

### 3. 过程打桩

对于一些没有返回值的函数，我们称为“过程”：

```Go
var DestroyResource = func() {
  fmt.Println("清理资源等工作")
}

```

打桩开始：

方式一：`StubFunc` 直接设置返回结果（当你想这个过程啥都不做时，可以这样）

```Go
stubs := gostub.StubFunc(&gomock_service.DestroyResource)
defer stubs.Reset()
```

方式二：`Stub` 还能设置具体逻辑

```Go
stubs := gostub.Stub(&gomock_service.DestroyResource, func() {
      // do sth
    })
defer stubs.Reset()
```

### 4. 第三方库打桩

很多第三方库的函数（注意，是函数，不是某个对象的某个成员方法），我们会经常使用，而在单元测试时不是我们的关注点，或者想他报错等，就可以选择打桩。

1. 假如，我想打桩json的序列化和反序列化函数，那么，先在`adapter`包下定义`json.go`文件，然后声明对象：
    ```Go
    package adapter

    import (
      "encoding/json"
    )

    var Marshal = json.Marshal
    var UnMarshal = json.Unmarshal 
    ```

2. 单元测试中，就可以直接使用`gostub.StubFunc`来进行打桩了：
    ```Go
    // given
    var mikeStr = `{"name":"Jasper", "age":18}`
    stubs := gostub.StubFunc(&adapter.Marshal, []byte(mikeStr), nil)
    defer stubs.Reset()

    stu := &entity.Student{Name: "Mike", Age: 18}

    // when
    res, err := stu.Print()

    // then
    assert.Equal(t, "{\"name\":\"Jasper\", \"age\":18}", res)
    assert.Nil(t, err)
    ```

## Demo示例

[https://github.com/androidjp/go-mock-best-practice/tree/1_gomock_2_gostub](https://github.com/androidjp/go-mock-best-practice/tree/1_gomock_2_gostub)

# goconvey更优化的断言

## 需求

- 更优雅地写测试用例

- 更好地可视化界面，实时更新当前所有测试情况和覆盖率

## 安装与配置

```Go
go get -v -u github.com/smartystreets/goconvey
```

## 如何跑测试

cd到测试文件所在目录，执行：

```纯文本
go test -v 
```

或者，cd到项目根目录，全跑整个项目的测试：

```Go
go test ./... -v -cover
```

或者执行以下命令，弹出web页面（默认：[http://127.0.0.1:8080](http://127.0.0.1:8080)）：

```纯文本
goconvey
```

其中，`goconvey -help` 能出来相关的命令行选项说明，一般常用是这么写：

```纯文本
goconvey -port 8112 -excludedDirs "vendor,mock,proto,conf"
```

表示：web页面在 [http://127.0.0.1:8112](http://127.0.0.1:8112)，并且忽略当前上下文目录下的vendor目录、mock目录、proto目录。

图示：

![](/images/20210323/2.png)

## 格式规范

```Go
func TestXxxService_XxxMethod(t *testing.T) {
  Convey("should return 情况A", t, func() {
    Convey("when 某逻辑 return `xxxx`", func() {
            // given
            ..........各种准备逻辑（mock、stub、声明、初始化、造数据等）
            // when
            res, err := xxxService.XxxMethod(....)
            // then
            So(res, ShouldEqual, "情况A")
        })
        Convey("when 传入了参数 keyA=`valA`, keyB=`valB`", func() {

        })
    })

    Convey("should return 情况B", t, func() {
    Convey("when ..................", func() {

        })
    })
}
```

如：

```纯文本
........
Convey("should return `解析响应体失败：response is empty`", t, func() {
    Convey("when proxy response with statusCode 200 and empty body is returned", func() {
........
```

## IDE 快速生成单元测试代码

![](/images/20210323/3.png)

以下的本人实践使用的模板：

```Go
Convey("should $FIRST$", t, func() {
    Convey("when $SECOND$", func() {
        //---------------------------------------
        // given
        //---------------------------------------
        $END$
        //---------------------------------------
        // when
        //---------------------------------------
        
        //---------------------------------------
        // then
        //---------------------------------------

    })
})
```

设置完毕后，写测试时，直接键入 `swg`然后点击 `tab`键，即可生成这段模板代码。

> 注意：测试代码当然需要引入convey库：`import . "github.com/smartystreets/goconvey/convey"`

## Demo示例

[androidjp/go-mock-best-practice](https://github.com/androidjp/go-mock-best-practice/tree/1_gomock_3_gostub_gocovey)

# GoMonkey

## 需求

- 为一个函数打桩

- 为一个过程打桩

- 为一个方法打桩

- 特殊场景：桩中桩的一个案例

## 使用场景

- 基本场景：为一个函数打桩

- 基本场景：为一个过程打桩

- 基本场景：为一个方法打桩

- 特殊场景：桩中桩的一个案例

## 局限

- Monkey不是线程安全的，不要将Monkey用于并发的测试

- 对inline函数打桩无效（一般需要：通过命令行参数`-gcflags=-l`禁止inline）
    ```Go
    // 像这种函数，很简单短小，在源码层面来看时有函数结构的，但是编译后却不具备函数的性质。
    func IsEqual(a, b string) bool {
        return a == b
    }
    ```

- Monkey只能为首字母大写的方法/函数打桩（当然，这样其实更符合编码规范）。

- API不够简洁优雅，同时不支持多次调用桩函数（方法）而呈现不同行为的复杂情况。

## 安装

```Bash
go get -v bou.ke/monkey
```

## 运行单元测试

方式一：命令行运行

```Bash
go test -gcflags=-l -v
```

方式二：IDE 运行：在 Go tool Arguments 栏加上这一个选项：`-gcflags=-l`



注意：在运行测试时，可能出现运行报错的问题，

原因：由于它是在运行时替换了函数的指针，所以如果遇到一些简单的函数，例如 rand.Int63n 和 time.Now，编译器可能会直接将这种函数内联到调用实际发生的代码处并不会调用原有的方法，所以使用这种方式往往需要我们在测试时额外指定 -gcflags=-l 禁止编译器的内联优化。

这时，几种运行方式：

1. 命令行运行
    ```Bash
    go test -gcflags=-l -v
    ```
2. IDE运行，加上`-gcflags=-l`
    ![](/images/20210323/4.png)


## 关键用法

### 1. 函数打桩

1. 假设目前有这样一个函数 `Exec`：
    ```Go
    package service

    func Exec(cmd string, args ...string) (string, error) {
        //...........
    }
    ```

2. 我们直接可以使用`monkey.Patch`将其打桩，不需要声明什么变量：
    ```Go
    // 打桩
    guard := Patch(service.Exec, func(cmd string, args ...string) (string, error) {
        return "sss", nil
    })
    defer guard.Unpatch()
    // 调用
    output, err := service.Exec("cmd01", "--conf", "conf/app_test.yaml")
    ```

### 2. 过程打桩

和函数一样，相较于`gostub`好的一点，就是不需要声明变量去指向这个函数，从而减少业务代码的修改。

1. 假设有这样一个过程：
    ```Go
    func InternalDoSth(mData map[string]interface{}) {
        mData["keyA"] = "valA"
    }
    ```
2. 一样方式进行打桩
    ```Go
    patchGuard := Patch(service.InternalDoSth, func(mData map[string]interface{}) {
        mData["keyA"] = "valB"
    })
    defer patchGuard.Unpatch()

    ..............
    ```

### 3. 方法打桩

注意：只能打Public方法的桩，也就是首字母大写的方法

1. 假设有这样一个类以及它的方法定义：
    ```Go
    type Etcd struct {
    }

    // 成员方法
    func (e *Etcd) Get(id int) []string {
        names := make([]string, 0)
        switch id {
        case 0:
            names = append(names, "A")
        case 1:
            names = append(names, "B")
        }
        return names
    }

    func (e *Etcd) Save(vals []string) (string, error) {
        return "存储DB成功", nil
    }

    func (e *Etcd) GetAndSave(id int) (string, error) {
        vals := e.Get(id)
        if vals[0] == "A" {
            vals[0] = "C"
        }
        return e.Save(vals)
    }
    ```

2. 通过`PatchInstanceMethod`即可打桩，然后直接调用:
    ```Go
    // 打桩
    var e = &service.Etcd{}
    guard := PatchInstanceMethod(reflect.TypeOf(e), "Get", func(e *service.Etcd, id int) []string {
        return []string{"Jasper"}
    })
    defer guard.Unpatch()

    // 调用
    res := e.Get(1)
    ```

3. 当我想要一个测试用例里打多个成员方法的桩，这样即可：
    ```Go
    var e = &service.Etcd{}
    // stub Get
    theVals := make([]string, 0)
    theVals = append(theVals, "A")
    PatchInstanceMethod(reflect.TypeOf(e), "Get", func(e *service.Etcd, id int) []string {
        return theVals
    })
    // stub Save
    PatchInstanceMethod(reflect.TypeOf(e), "Save", func(e *service.Etcd, vals []string) (string, error) {
        return "", errors.New("occurs error")
    })

    // 一键删除所有补丁
    defer UnpatchAll()

    .............
    ```

### 4. 配合gomock（桩中桩）

当我需要mock一个接口，并且，重新定义这个mock对象的某个方法时，使用。

详情看demo例子`repo_test.go`

## Demo示例

[androidjp/go-mock-best-practice](https://github.com/androidjp/go-mock-best-practice/tree/3_monkey)

## 坑

由于使用GoMonkey Patch后导致GoConvey命令不能正常运行测试用例

解决方案：
[https://blog.csdn.net/scun_cg/article/details/88395041](https://blog.csdn.net/scun_cg/article/details/88395041)



# 原生http服务接口测试

## 需求

- 原生`net/http`写的web服务，http接口需要单元测试。

## 例子

假设我们不用任何web框架（Gin、Beego、Echo等），使用原生的golang `net/http` 库编写一个RESTful接口服务，这样写：

1. 定义controller
    ```Go
    var (
      instanceDemoController *DemoController
      initDemoControllerOnce sync.Once
    )

    type DemoController struct {
    }

    func GetDemoController() *DemoController {
      initDemoControllerOnce.Do(func() {
        instanceDemoController = &DemoController{}
      })
      return instanceDemoController
    }

    func (d *DemoController) GetMessage(w http.ResponseWriter, r *http.Request) {
      r.ParseForm()       // 解析参数，默认是不会解析的
      fmt.Println(r.Form) // 这些信息是输出到服务器端的打印信息
      fmt.Println("path", r.URL.Path)
      fmt.Println("scheme", r.URL.Scheme)
      fmt.Println(r.Form["url_long"])
      for k, v := range r.Form {
        fmt.Println("key:", k)
        fmt.Println("val:", strings.Join(v, ""))
      }
      fmt.Fprintf(w, "Hello Mike!") // 这个写入到 w 的是输出到客户端的
    }

    ```

2. main方式直接使用http包的`HandleFunc`和`ListenAndServe`方法即可完成监听并启动服务：
    ```Go
    func main() {
      // 设置访问的路由
      http.HandleFunc("/message", controller.GetDemoController().GetMessage)
      // 设置监听的端口
      fmt.Println("Start listening 9090, 尝试请求：http://localhost:9090/message?keyA=valA&url_long=123456")
      if err := http.ListenAndServe(":9090", nil); err != nil {
        log.Fatal("ListenAdnServe: ", err)
      }
    }
    ```
    那，如果我要单元测试测这个接口怎么办？原生的`net/http/httptest` 就能帮到你：
    ```Go
    //---------------------------------------
    // given
    //---------------------------------------
    demoCtrl := &controller.DemoController{}
    ts := httptest.NewServer(http.HandlerFunc(demoCtrl.GetMessage))
    defer ts.Close()

    //---------------------------------------
    // when
    //---------------------------------------
    resp, err := http.Get(ts.URL)
    defer resp.Body.Close()
    bodyBytes, err := ioutil.ReadAll(resp.Body)

    //---------------------------------------
    // then
    //---------------------------------------
    assert.Nil(t, err)
    assert.Equal(t, "Hello Mike!", string(bodyBytes))
    ```

## Demo示例

[androidjp/go-mock-best-practice](https://github.com/androidjp/go-mock-best-practice/tree/2_web_2_raw_http_httptest)

# Gin接口测试

gin官方文档：[https://github.com/gin-gonic/gin#quick-start](https://github.com/gin-gonic/gin#quick-start)

## 需求

- 测试基于Gin框架写的API接口。

## 安装

我们可以使用原生的httptest来测试接口，当然也可以使用其他的库，如：[apitest](https://apitest.dev/)

```Go
// gin
go get -v -u github.com/gin-gonic/gin
// testify 断言库
go get -v -u github.com/stretchr/testify

```

## 原生测试写法

1. 首先，我们DemoController代码逻辑基本没变，只是需要采用`*gin.Context` ，启动函数只是变得更简洁而已：
    ```Go
    func main() {
      // 设置访问的路由
      r := gin.Default()
      r.GET("/message", controller.GetDemoController().GetMessage)

      // 设置监听的端口
      fmt.Println("Start listening 9090, 尝试请求：http://localhost:9090/message?keyA=valA&url_long=123456")
      if err := r.Run(":9090"); err != nil {
        log.Fatal("ListenAdnServe: ", err)
      }
    }
    ```

2. 那么，测试用例写法，也只是前期准备gin测试环境的逻辑有所不同罢了：
    ```Go
    //---------------------------------------
    // given
    //---------------------------------------
    gin.SetMode(gin.TestMode)
    router := gin.New()
    demoCtrl := &controller.DemoController{}
    //待测试的接口
    router.GET("/message", demoCtrl.GetMessage)

    //---------------------------------------
    // when
    //---------------------------------------
    // 构建返回值
    w := httptest.NewRecorder()
    // 构建请求
    req, _ := http.NewRequest("GET", "/message?keyA=valA&url_long=123456", nil)
    //调用请求接口
    router.ServeHTTP(w, req)

    resp := w.Result()
    body, err := ioutil.ReadAll(resp.Body)

    //---------------------------------------
    // then
    //---------------------------------------
    assert.Nil(t, err)
    assert.Equal(t, "Hello Mike!", string(body))
    ```

## Demo示例

[androidjp/go-mock-best-practice](https://github.com/androidjp/go-mock-best-practice/tree/2_web_3_gin_httptest)

# apitest

## 需求

- 对golang原生http或者Gin等框架下的RESTFul API接口进行更简洁的测试

## 安装

官网：[https://apitest.dev/](https://apitest.dev/)

github地址：[https://github.com/steinfletcher/apitest](https://github.com/steinfletcher/apitest)

`go get -u github.com/steinfletcher/apitest`

## 用apitest测试Gin接口

各种apitest详细用法可以参考官方文档，这里只说说也前面使用原生httptest的最大区别：更简洁了。

```Go
//---------------------------------------
// given
//---------------------------------------
gin.SetMode(gin.TestMode)
router := gin.New()
demoCtrl := &controller.DemoController{}
//待测试的接口
router.GET("/message", demoCtrl.GetMessage)

//---------------------------------------
// when then
//---------------------------------------
apitest.New().
  Handler(router).
  Getf("/message?keyA=%s&url_long=1%s", "valA", "123456").
  Header("Client-Type", "pc").
  Cookie("sid", "id001").
  JSON(nil).
  Expect(t).
  Status(http.StatusOK).
  Assert(jsonPath.Equal(`$.code`, float64(2000))).
  Assert(jsonPath.Equal(`$.msg`, "Hello Mike!")).
  Body(`{"code":2000,"msg":"Hello Mike!"}`).
  End()
```

## Demo示例

[androidjp/go-mock-best-practice](https://github.com/androidjp/go-mock-best-practice/tree/2_web_4_gin_apitest)



# Beego 接口测试

参考文章：[https://blog.csdn.net/qq_38959696/article/details/106567212](https://blog.csdn.net/qq_38959696/article/details/106567212)



# SqlMock使用

github：[https://github.com/DATA-DOG/go-sqlmock](https://github.com/DATA-DOG/go-sqlmock)

## 需求

- 测试自己写的sql脚本等与DB存取息息相关的细节逻辑有没有问题

## 安装

```Go
go get -v -u github.com/DATA-DOG/go-sqlmock
```

## 关键用法

### 情况一：直接将db对象作为入参

假设有这样的一个函数，直接传入的是`*sql.DB`：

```Go
func (d *DemoService) AddStudentDirectly(db *sql.DB, name string) (stu *entities.Student, err error) {
  // 启动事务
  tx, err := db.Begin()
  if err != nil {
    return nil, err
  }

  defer func() {
    switch err {
    case nil:
      err = tx.Commit()
    default:
      tx.Rollback()
    }
  }()

  // 1. 先新增一个学生信息
  result, err := db.Exec("insert into students(name) values(?)", name)
  if err != nil {
    return
  }
  id, err := result.LastInsertId()
  if err != nil {
    return
  }
  // 2. 然后，给教室1 添加这个学生
  if _, err = db.Exec("insert into classroom_1(stu_id) values(?)", id); err != nil {
    return
  }
  stu = &entities.Student{ID: id, Name: name}
  return
}
```

以上函数做的事情：开事务、插入students表、将得到的id，插入另一长classroom_1表，最终提交事务。

这种情况，只需要想办法mock掉这个`db *sql.DB`对象即可：

1. sqlmock.New()得到mock掉了的db对象，以及mock记录器对象；
    ```Go
    db, mock, err := sqlmock.New()
    if err != nil {
      t.Fatalf("an error '%s' was not expected when opening a stub database connection", err)
    }
    defer db.Close()
    ```

2. 设置即将要发生什么DB操作，并且设置将返回什么值：
    ```Go
    // 1) 首先是会 开启事务
    mock.ExpectBegin()
    // 2) 然后是插入students表，最终返回id=1, 影响行数=1
    mock.ExpectExec(regexp.QuoteMeta(`insert into students(name) values(?)`)).WithArgs("mike").WillReturnResult(sqlmock.NewResult(1, 1))
    // 3) 插入classroom_1表
    mock.ExpectExec(regexp.QuoteMeta("insert into classroom_1(stu_id) values(?)")).WithArgs(1).WillReturnResult(sqlmock.NewResult(1, 1))
    // 4) 提交事务
    mock.ExpectCommit()
    ```

3. 将mock了的db对象，作为入参：
    ```Go
    ...........
    stu, err := svr.AddStudentDirectly(db, "mike")
    ........... 
    ```

### 情况二：有自己的repository层对象，去封装db的相关操作

一般我们写项目代码，职责单一，分层明确，很多时候，会是另一种写法：repository层的某个XxxRepository类拥有DB连接对象，然后，有一系列对应的DB相关操作：

```Go
type MySQLRepository struct {
  db *sql.DB
}

func NewMySQLRepository() *MySQLRepository {
  db, err := sql.Open("mysql", "root:root@tcp(192.168.200.128:3307)/test?charset=utf8mb4")
  if err != nil {
    panic(err)
  }
  db.SetConnMaxLifetime(time.Minute * 2)
  db.SetMaxOpenConns(10)
  db.SetMaxIdleConns(10)

  return &MySQLRepository{
    db: db,
  }
}

func (m *MySQLRepository) CreateStudent(name string) (stu *entities.Student, err error) {
    ..........................
} 
```

此时，如果想要更好地达到mock的目的，那么，需要配置gostub框架：

1. 首先，源码需要稍微做一些调整，加上adapter适配一下`sql.Open`为`adapter.Open`：
    ```Go
    package adapter

    import "database/sql"

    var Open = sql.Open
    ```

2. 然后，同样是通过sqlmock定义mock的db对象：
    ```Go
    db, mock, err := sqlmock.New()
    if err != nil {
      t.Fatalf("an error '%s' was not expected when opening a stub database connection", err)
    }
    defer db.Close()

    mock.ExpectBegin()
    mock.ExpectExec(regexp.QuoteMeta(`insert into students(name) values(?)`)).WithArgs("mike").WillReturnResult(sqlmock.NewResult(1, 1))
    mock.ExpectExec(regexp.QuoteMeta("insert into classroom_1(stu_id) values(?)")).WithArgs(1).WillReturnResult(sqlmock.NewResult(1, 1))
    mock.ExpectCommit()
    ```

3. 然后，gostub让adapter.Open打桩，让其返回我们mock的db对象：
    ```Go
    stubs := gostub.StubFunc(&adapter.Open, db, nil)
    defer stubs.Reset()
    ```

4. 最终实际跑逻辑进行测试：
    ```Go
    sqlRepository := repository.NewMySQLRepository()
    student, err := sqlRepository.CreateStudent("mike") 
    ```
    当然，对于不同的ORM框架，有一些不同的封装逻辑，获取到的db操作对象也可能不是`*sql.DB` 类型，这里详细可以看Demo示例，这里目前只实践了MySQL gorm和xorm两个ORM框架的mock实践。

## Demo示例

- MySQL 原生 sql-driver 的mock：[https://github.com/androidjp/go-mock-best-practice/tree/4_db_1_sqlmock_sqldriver](https://github.com/androidjp/go-mock-best-practice/tree/4_db_1_sqlmock_sqldriver)

- MySQL gorm ORM框架mock：[https://github.com/androidjp/go-mock-best-practice/tree/4_db_2_sqlmock_gorm](https://github.com/androidjp/go-mock-best-practice/tree/4_db_2_sqlmock_gorm)

- MySQL xorm ORM框架mock：[https://github.com/androidjp/go-mock-best-practice/tree/4_db_3_sqlmock_xorm](https://github.com/androidjp/go-mock-best-practice/tree/4_db_3_sqlmock_xorm)


# 其他

## 一、Gin的`*gin.Context` 怎么mock？

1. 首先，在测试文件中定义一个MockResponseWriter结构体：
    ```Go
    type MockResponseWriter struct {
    }

    func (m *MockResponseWriter) Header() http.Header {
      h := make(map[string][]string)
      h["Client-Type"] = []string{"wps-pc"}
      return h
    }

    func (m *MockResponseWriter) Write([]byte) (int, error) {
      return 0, nil
    }

    func (m *MockResponseWriter) WriteHeader(statusCode int) {

    }
    ```

2. 利用gin.CreateTestContext，来构造整个gin.Context：
    ```Go
    mockGinContext, _ := gin.CreateTestContext(&MockResponseWriter{})
    mockGinContext.Request = &http.Request{}
    // mock request header
    mockGinContext.Request.Header = make(map[string][]string)
    mockGinContext.Request.Header["Client-Type"] = []string{"wps-pc"}
    mockGinContext.Request.Header["Client-Chan"] = []string{"00050.88888888"}
    mockGinContext.Request.Header["Client-Ver"] = []string{"1.0.1"}
    mockGinContext.Request.Header["X-Forwarded-Host"] = []string{"test.com"}
    mockGinContext.Request.URL = &url.URL{Path: "/api/v2/proofread"}
    mockGinContext.Set("uid", "123123123")

    // mock request body
    mockGinContext.Request.Body = ioutil.NopCloser(bytes.NewReader([]byte("{\"key\":\"val\",\"userid\":123123}")))
    ```

3. ok，这个`mockGinContext`可以被我们拿来作为参数使用了。

## 二、[POST data using the Content-Type multipart/form-data](https://stackoverflow.com/questions/20205796/post-data-using-the-content-type-multipart-form-data)

In short, you'll need to use the mime/multipart package to build the form.

sample代码：

```go
 package main

import (
    "bytes"
    "fmt"
    "io"
    "mime/multipart"
    "net/http"
    "net/http/httptest"
    "net/http/httputil"
    "os"
    "strings"
)

func main() {

    var client *http.Client
    var remoteURL string
    {
        //setup a mocked http client.
        ts := httptest.NewTLSServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            b, err := httputil.DumpRequest(r, true)
            if err != nil {
                panic(err)
            }
            fmt.Printf("%s", b)
        }))
        defer ts.Close()
        client = ts.Client()
        remoteURL = ts.URL
    }

    //prepare the reader instances to encode
    values := map[string]io.Reader{
        "file":  mustOpen("main.go"), // lets assume its this file
        "other": strings.NewReader("hello world!"),
    }
    err := Upload(client, remoteURL, values)
    if err != nil {
        panic(err)
    }
}

func Upload(client *http.Client, url string, values map[string]io.Reader) (err error) {
    // Prepare a form that you will submit to that URL.
    var b bytes.Buffer
    w := multipart.NewWriter(&b)
    for key, r := range values {
        var fw io.Writer
        if x, ok := r.(io.Closer); ok {
            defer x.Close()
        }
        // Add an image file
        if x, ok := r.(*os.File); ok {
            if fw, err = w.CreateFormFile(key, x.Name()); err != nil {
                return
            }
        } else {
            // Add other fields
            if fw, err = w.CreateFormField(key); err != nil {
                return
            }
        }
        if _, err = io.Copy(fw, r); err != nil {
            return err
        }

    }
    // Don't forget to close the multipart writer.
    // If you don't close it, your request will be missing the terminating boundary.
    w.Close()

    // Now that you have a form, you can submit it to your handler.
    req, err := http.NewRequest("POST", url, &b)
    if err != nil {
        return
    }
    // Don't forget to set the content type, this will contain the boundary.
    req.Header.Set("Content-Type", w.FormDataContentType())

    // Submit the request
    res, err := client.Do(req)
    if err != nil {
        return
    }

    // Check the response
    if res.StatusCode != http.StatusOK {
        err = fmt.Errorf("bad status: %s", res.Status)
    }
    return
}

func mustOpen(f string) *os.File {
    r, err := os.Open(f)
    if err != nil {
        panic(err)
    }
    return r
}
```

