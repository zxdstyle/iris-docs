---
title: Session
menu:
    docs:
        parent: 基本功能
        weight: 3
---


# Sessions

使用应用程序时，请先将其打开，进行一些更改，然后再将其关闭。 这很像一个会话。 电脑知道您是谁。 它知道何时启动应用程序以及何时终止。 但是在Internet上存在一个问题：Web服务器不知道您是谁或做什么，因为HTTP地址无法保持状态。

Session 变量通过存储要在多个页面上使用的用户信息（例如用户名，喜欢的颜色等）来解决此问题。 默认情况下，会话变量将持续到用户关闭浏览器为止。

所以, Session 变量保存有关一个用户的信息，并且可用于一个应用程序中的所有页面。

> **提示**: 如果您需要一个永久存储器，你可以试试存储在 [数据库](/docs/base/Sessions-database).

-------

Iris 有自己的 session 实现，sessions 管理器在 [Iris/sessions](https://github.com/kataras/iris/tree/master/sessions) 子包。您需要导入此包才能使用 HTTP 会话。

Session 是通过使用 `New` 包级别函数创建的 `Sessions` 对象的` Start` 函数启动的。该函数将返回一个`Session`。

Session 变量是使用 `Session.Set` 方法设置，并可以通过 `Session.Get` 及其相关方法进行检索。 要删除单个变量，请使用 `Session.Delete`。 要删除整个会话并使它们无效，请使用 `Session.Destroy` 方法。

## 所有方法

sessions 管理器包级别的函数 `New` 创建。

```go
import "github.com/kataras/iris/v12/sessions"

sess := sessions.New(sessions.Config{Cookie: "cookieName", ...})
```

其中 `Config` 看起来是这样。

```go
SessionIDGenerator func(iris.Context) string

// Defaults to "irissessionid".
Cookie string

CookieSecureTLS bool

// Defaults to false.
AllowReclaim bool

// Defaults to nil.
Encode func(cookieName string, value interface{}) (string, error)
// Defaults to nil.
Decode func(cookieName string, cookieValue string, v interface{}) error
// Defaults to nil.
Encoding Encoding

// Defaults to infinitive/unlimited life duration(0).
Expires time.Duration

// Defaults to false.
DisableSubdomainPersistence bool
```

返回 `Sessions` 指针，拥有以下方法。

```go
Start(ctx iris.Context,
    cookieOptions ...iris.CookieOption) *Session

Handler(cookieOptions ...iris.CookieOption) iris.Handler

Destroy()
DestroyAll()
DestroyByID(sessID string)
OnDestroy(callback func(sid string))

ShiftExpiration(ctx iris.Context,
    cookieOptions ...iris.CookieOption) error
UpdateExpiration(ctx iris.Context, expires time.Duration,
    cookieOptions ...iris.CookieOption) error

UseDatabase(db Database)
```

其中 `CookieOption` 只是一个 `func(*http.Cookie)` 允许自定义 cookie 的属性。

`Start` 方法返回一个 `Session` 指针值，该指针值导出自己的方法在每个会话中工作。

```go
func (ctx iris.Context) {
    session := sess.Start(ctx)
     .ID() string
     .IsNew() bool

     .Set(key string, value interface{})
     .SetImmutable(key string, value interface{})
     .GetAll() map[string]interface{}
     .Len() int
     .Delete(key string) bool
     .Clear()

     .Get(key string) interface{}
     .GetString(key string) string
     .GetStringDefault(key string, defaultValue string) string
     .GetInt(key string) (int, error)
     .GetIntDefault(key string, defaultValue int) int
     .Increment(key string, n int) (newValue int)
     .Decrement(key string, n int) (newValue int)
     .GetInt64(key string) (int64, error)
     .GetInt64Default(key string, defaultValue int64) int64
     .GetFloat32(key string) (float32, error)
     .GetFloat32Default(key string, defaultValue float32) float32
     .GetFloat64(key string) (float64, error)
     .GetFloat64Default(key string, defaultValue float64) float64
     .GetBoolean(key string) (bool, error)
     .GetBooleanDefault(key string, defaultValue bool) bool

     .SetFlash(key string, value interface{})
     .HasFlash() bool
     .GetFlashes() map[string]interface{}

     .PeekFlash(key string) interface{}
     .GetFlash(key string) interface{}
     .GetFlashString(key string) string
     .GetFlashStringDefault(key string, defaultValue string) string

     .DeleteFlash(key string)
     .ClearFlashes()

     .Destroy()
}

```


## 示例

在此示例中，我们仅允许经过身份验证的用户通过 `/secret` 查看加密的年龄消息。 要访问它，必须先访问 `/login` 以获取有效的会话 cookie，然后 hich 将成功登录。此外，他可以访问 `/logout` 撤消对我们加密消息的访问。

```go
// sessions.go
package main

import (
    "github.com/kataras/iris/v12"

    "github.com/kataras/iris/v12/sessions"
)

var (
    cookieNameForSessionID = "mycookiesessionnameid"
    sess                   = sessions.New(sessions.Config{Cookie: cookieNameForSessionID})
)

func secret(ctx iris.Context) {
    // 检查用户是否授权
    if auth, _ := sess.Start(ctx).GetBoolean("authenticated"); !auth {
        ctx.StatusCode(iris.StatusForbidden)
        return
    }

    // 打印加密消息
    ctx.WriteString("The cake is a lie!")
}

func login(ctx iris.Context) {
    session := sess.Start(ctx)

    // Authentication goes here
    // ...

    // Set user as authenticated
    session.Set("authenticated", true)
}

func logout(ctx iris.Context) {
    session := sess.Start(ctx)

    // Revoke users authentication
    session.Set("authenticated", false)
    // Or to remove the variable:
    session.Delete("authenticated")
    // Or destroy the whole session:
    session.Destroy()
}

func main() {
    app := iris.New()

    app.Get("/secret", secret)
    app.Get("/login", login)
    app.Get("/logout", logout)

    app.Listen(":8080")
}
```

```sh
$ go run sessions.go

$ curl -s http://localhost:8080/secret
Forbidden

$ curl -s -I http://localhost:8080/login
Set-Cookie: mysessionid=MTQ4NzE5Mz...

$ curl -s --cookie "mysessionid=MTQ4NzE5Mz..." http://localhost:8080/secret
The cake is a lie!
```

## 中间件

当您想在相同的请求处理程序和生命周期（处理程序，中间件链）中使用 `Session` 时，可以 （可选）将其注册为中间件，并使用包级的 `sessions.Get`  检索存储到上下文的 session。

`Sessions` 结构体值包含 `Handler` 方法，该方法可用于返回要注册为中间件的 `iris.Handler`。

```go
import "github.com/kataras/iris/v12/sessions"

sess := sessions.New(sessions.Config{...})

app := iris.New()
app.Use(sess.Handler())

app.Get("/path", func(ctx iris.Context){
    session := sessions.Get(ctx)
    // [use session...]
})
```

同样，如果会话管理器的 `Config.AllowReclaim` 为 `true`，那么您仍然可以在相同的请求生命周期中多次调用 `sess.Start`，而无需将其注册为中间件。

跳转到 [https://github.com/kataras/iris/tree/master/_examples/sessions](https://github.com/kataras/iris/tree/master/_examples/sessions) 查看 session 相关的详细示例

接下来继续讲解 [闪存消息](/docs/base/sessions-flash-messages)。
