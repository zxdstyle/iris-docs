---
title: 依赖注入
menu:
    docs:
        parent: 基本功能
        weight: 1
---

Iris 通过请求处理程序和基于返回值的服务器回复为依赖注入提供了一流的支持。

## 依赖注入

使用虹膜，你可以得到真正安全的绑定。它的速度非常快，接近原始处理程序的性能，因为我们在服务器上线之前就预先分配了必要的信息！

依赖项可以是函数或静态值。函数依赖项也可以接受以前注册的依赖项作为其输入参数。

示例代码:

```go
func printFromTo(from, to string) string { return "message" }

// [...]
app.ConfigureContainer(func (api *iris.APIContainer){
    api.Get("/{from}/{to}", printFromTo)
})
```

正如你在上面看到的 `iris.Context` 输入参数是完全可选的。Iris 非常聪明，可以毫无麻烦地将它绑定起来。

### 概述

要处理的路由最常见的情况是：
- 接受一个或多个路径参数并请求数据，即有效负载
- 返回响应 , 有效负载 (JSON, XML,...)

在上述情况下，新的 Iris 依赖注入特性比前一个特性快了大约 **33.2%**。这会进一步降低本地处理程序和具有依赖关系的处理程序之间的性能成本。这一点本身就为我们带来了安全性和性能方面的全新体验 `Party.ConfigureContainer(builder ...func(*iris.APIContainer)) *APIContainer` 方法，它返回诸如 `Handle(method, relativePath string, handlersFn ...interface{}) *Route` 和 `RegisterDependency`.

看看使用 Iris 时你的代码库有多干净：

```go
package main

import "github.com/kataras/iris/v12"

type (
    testInput struct {
        Email string `json:"email"`
    }

    testOutput struct {
        ID   int    `json:"id"`
        Name string `json:"name"`
    }
)

func handler(id int, in testInput) testOutput {
    return testOutput{
        ID:   id,
        Name: in.Email,
    }
}

func main() {
    app := iris.New()
    app.ConfigureContainer(func(api *iris.APIContainer) {
        api.Post("/{id:int}", handler)
    })
    app.Listen(":5000", iris.WithOptimizations)
}
```

你的眼睛不会骗你。看上去很输入，不是吗 ？`ctx.ReadJSON(&v)` 和 `ctx.JSON(send)` 没有 `错误` 处理。这是一个巨大的解脱，但如果您需要，您仍然可以控制这些，甚至来自依赖项的错误。`Party.ConfigureContainer()` 是一个新的字段和方法的快速列表：

```go
// Container 拥有该方具有依赖关系注入功能的DI Container。
// 使用它可以将函数或结构（控制器）手动转换为Handler。
Container *hero.Container
```

```go
// OnError为此方法的DI Hero容器及其处理程序（或控制器）添加错误处理程序。 
// errorHandler 处理任何可能发生并返回的错误
OnError(errorHandler func(iris.Context, error))
```

```go
// RegisterDependency 添加了一个依赖项。
// 值可以是单个结构值或函数。
// 遵守以下规则：
// * <T> {structValue}
// * func(accepts <T>)                                 returns <D> or (<D>, error)
// * func(accepts iris.Context)                        returns <D> or (<D>, error)
//
// 依赖项可以接受以前注册的依赖项并返回新的依赖项或更新的依赖项。
// * func(accepts1 <D>, accepts2 <T>)                  returns <E> or (<E>, error) or error
// * func(acceptsPathParameter1 string, id uint64)     returns <T> or (<T>, error)
//
// Usage:
//
// - RegisterDependency(loggerService{prefix: "dev"})
// - RegisterDependency(func(ctx iris.Context) User {...})
// - RegisterDependency(func(User) OtherResponse {...})
RegisterDependency(dependency interface{})

// UseResultHandler将结果处理程序添加到容器中。
// 结果处理程序可用于从请求处理程序注入返回的结构值//或替换默认呈现程序。
UseResultHandler(handler func(next iris.ResultHandler) iris.ResultHandler)
```

<details><summary>ResultHandler</summary>

```go
type ResultHandler func(ctx iris.Context, v interface{}) error
```
</details>

```go
// 使用方法与普通的 `Use` 相同，但它接受动态方法 `handlersFn` 作为其输入。
Use(handlersFn ...interface{})
// 与公共方法的方法相同，但它接受动态函数作为“handlersFn”输入。
Done(handlersFn ...interface{})
```

```go
// Handle same as a common Party's `Handle` but it accepts one or more "handlersFn" functions which each one of them
// can accept any input arguments that match with the Party's registered Container's `Dependencies` and
// any output result; like custom structs <T>, string, []byte, int, error,
// a combination of the above, hero.Result(hero.View | hero.Response) and more.
//
// 处理方式与普通方法的 `Handle` 相同，但接受一个或多个 `handlersFn` 方法，每个功能均由其处理
// 可以接受与该方法注册的容器的依赖相匹配的任何输入参数，并且任何输出结果；比如自定义结构体<T>，字符串，[]byte，int，错误，
// 以上各项的组合 hero.Result（hero.View | hero.Response）等。

// 因此，从 hero 处理程序开始，甚至不需要接受 `Context`，
// 这是很常见的，如果不手动调用 "handlersFn" ，它将自动调用 `ctx.Next()`。
// 停止执行，而不继续执行下一个 `handlersFn` , 最终开发人员应该输出错误并返回 `iris.ErrStopExecution`。
Handle(method, relativePath string, handlersFn ...interface{}) *Route

// Get 注册一条 GET 路由, 类似 `Handle("GET", relativePath, handlersFn....)`.
Get(relativePath string, handlersFn ...interface{}) *Route
// and so on...
```

以下是可立即用作输入参数的内置依赖项列表：

| Type | Maps To |
|------|:---------|
| [*mvc.Application](https://pkg.go.dev/github.com/kataras/iris/v12/mvc?tab=doc#Application) | 当前 MVC 应用 |
| [iris.Context](https://pkg.go.dev/github.com/kataras/iris/v12/context?tab=doc#Context) | 当前 Iris 上下文 |
| [*sessions.Session](https://pkg.go.dev/github.com/kataras/iris/v12/sessions?tab=doc#Session) | 当前 Iris Session |
| [context.Context](https://golang.org/pkg/context/#Context) | [ctx.Request().Context()](https://golang.org/pkg/net/http/#Request.Context) |
| [*http.Request](https://golang.org/pkg/net/http/#Request) | `ctx.Request()` |
| [http.ResponseWriter](https://golang.org/pkg/net/http/#ResponseWriter) | `ctx.ResponseWriter()` |
| [http.Header](https://golang.org/pkg/net/http/#Header) | `ctx.Request().Header` |
| [time.Time](https://golang.org/pkg/time/#Time) | `time.Now()` |
| `string`, | |
| `int, int8, int16, int32, int64`, | |
| `uint, uint8, uint16, uint32, uint64`, | |
| `float, float32, float64`, | |
| `bool`, | |
| `slice` | [Path Parameter](https://github.com/kataras/iris/wiki/Routing-path-parameter-types) |
| Struct | [Request Body](https://github.com/kataras/iris/tree/master/_examples/request-body) of `JSON`, `XML`, `YAML`, `Form`, `URL Query`, `Protobuf`, `MsgPack` |

### Request & Response & Path Parameters

**1.** 为客户端的请求主体和服务器的响应声明Go类型。

```go
type (
	request struct {
		Firstname string `json:"firstname"`
		Lastname  string `json:"lastname"`
	}

	response struct {
		ID      uint64 `json:"id"`
		Message string `json:"message"`
	}
)
```

**2.** 创建路由处理器

路径参数和请求主体是自动绑定的。
- **id uint64** 绑定 "id:uint64"
- **input request** 绑定到客户端请求数据，如JSON

```go
func updateUser(id uint64, input request) response {
	return response{
		ID:      id,
		Message: "User updated successfully",
	}
}
```

**3.** 按路由组配置容器并注册路由。

```go
app.Party("/user").ConfigureContainer(container)

func container(api *iris.APIContainer) {
    api.Put("/{id:uint64}", updateUser)
}
```

**4.** 模拟 [客户端](https://curl.haxx.se/download.html) 向服务器发送数据并显示响应的请求。

```sh
curl --request PUT -d '{"firstanme":"John","lastname":"Doe"}' http://localhost:8080/user/42
```

```json
{
    "id": 42,
    "message": "User updated successfully"
}
```

### 自定义 Preflight

在继续下一节注册依赖项之前，您可能需要了解如何在发送给客户端之前通过 `iris.Context` 自定义响应。

服务器将在响应发送到客户端之前自动对函数的输出结构值执行 `Preflight(iris.Context) error` 方法。

例如，您希望根据处理程序中的自定义逻辑触发不同的HTTP状态代码，并在发送到客户端之前修改值（响应体）。您的响应类型应该包含如下所示的 `Preflight` 方法。
```go
type response struct {
	ID      uint64 `json:"id,omitempty"`
	Message string `json:"message"`
	Code    int    `json:"code"`
	Timestamp int64 `json:"timestamp,omitempty"`
}

func (r *response) Preflight(ctx iris.Context) error {
	if r.ID > 0 {
		r.Timestamp = time.Now().Unix()
	}

	ctx.StatusCode(r.Code)
	return nil
}
```

现在，每个返回`*response`值的处理程序都将自动调用 `response.Preflight` 方法。

```go
func deleteUser(db *sql.DB, id uint64) *response {
    // [...custom logic]

    return &response{
        Message: "User has been marked for deletion",
        Code: iris.StatusAccepted,
    }
}
```

如果您注册路由并触发一个请求，您应该会看到这样的输出，时间戳被填充，客户端将收到的响应的HTTP状态码是202（status Accepted）。

```json
{
  "message": "User has been marked for deletion",
  "code": 202,
  "timestamp": 1583313026
}
```

### 注册依赖

**1.** 导入包与数据库交互。
 go-sqlite3 包是数据库 [SQLite](https://www.sqlite.org/index.html) 的驱动程序。

```go
import "database/sql"
import _ "github.com/mattn/go-sqlite3"
```

**2.** 配置容器 ([see above](#request--response--path-parameters)), 注册依赖. 处理程序需要 `*sql.DB` 数据库实例。

```go
localDB, _ := sql.Open("sqlite3", "./foo.db")
api.RegisterDependency(localDB)
```

**3.** 注册一个用于创建用户的路由

```go
api.Post("/{id:uint64}", createUser)
```

**4.** 创建用户

处理程序接受一个数据库和一些客户端请求数据，如JSON、Protobuf、Form、URL查询等等。它返回一个响应。

```go
func createUser(db *sql.DB, user request) *response {
    // [custom logic using the db]
    userID, err := db.CreateUser(user)
    if err != nil {
        return &response{
            Message: err.Error(),
            Code: iris.StatusInternalServerError,
        }
    }

	return &response{
		ID:      userID,
		Message: "User created",
		Code:    iris.StatusCreated,
	}
}
```

**5.** 模拟 [客户端](https://curl.haxx.se/download.html) 创建用户.

```sh
# JSON
curl --request POST -d '{"firstname":"John","lastname":"Doe"}' \
--header 'Content-Type: application/json' \
http://localhost:8080/user
```

```sh
# Form (multipart)
curl --request POST 'http://localhost:8080/users' \
--header 'Content-Type: multipart/form-data' \
--form 'firstname=John' \
--form 'lastname=Doe'
```

```sh
# Form (URL-encoded)
curl --request POST 'http://localhost:8080/users' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'firstname=John' \
--data-urlencode 'lastname=Doe'
```

```sh
# URL Query
curl --request POST 'http://localhost:8080/users?firstname=John&lastname=Doe'
```

响应: 

```json
{
    "id": 42,
    "message": "User created",
    "code": 201,
    "timestamp": 1583313026
}
```

## 返回值

- 如果返回值是 `string` ，那么它将把该字符串作为响应的主体发送。
- 如果它是一个 `int`，那么它将把它作为状态码发送。
- 如果它是一个 `error`，那么它将设置一个错误请求，并将该错误作为其原因。
- 如果是 `error`和 `int`，则错误代码是输出的整数，而不是400（错误请求）。
- 如果它是一个自定义的 `struct`，那么当尚未设置响应类型时，它将作为JSON发送。
- 如果是自定义的 `struct` 和 `string`，那么第二个输出值 string 就是内容类型，依此类推。

| Type | Replies to |
|------|:---------|
| string | body |
| string, string | content-type, body |
| string, int | body, status code |
| int | status code |
| int, string | status code, body |
| error | if not nil, bad request |
| any, bool | if false then fires not found |
| <Τ> | JSON body |
| <Τ>, string | body, content-type |
| <Τ>, error | JSON body or bad request |
| <Τ>, int | JSON body, status code |
| [Result](https://godoc.org/github.com/kataras/iris/hero#Result) | calls its `Dispatch` method |
| [PreflightResult](https://godoc.org/github.com/kataras/iris/hero#PreflightResult) | calls its `Preflight` method |

>  `<T>` 指任意结构体值

```go
type response struct {
	ID      uint64 `json:"id,omitempty"`
	Message string `json:"message"`
	Code    int    `json:"code"`
	Timestamp int64 `json:"timestamp,omitempty"`
}

func (r *response) Preflight(ctx iris.Context) error {
	if r.ID > 0 {
		r.Timestamp = time.Now().Unix()
	}

	ctx.StatusCode(r.Code)
	return nil
}

func deleteUser(db *sql.DB, id uint64) *response {
    // [...custom logic]

    return &response{
        Message: "User has been marked for deletion",
        Code: iris.StatusAccepted,
    }
}
```

稍后，您将看到这些知识将如何帮助您使用MVC架构模式（Iris为其提供了出色的API）构建应用程序。
