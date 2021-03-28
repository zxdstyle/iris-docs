---
title: 闪存消息
menu:
    docs:
        parent: 基本功能
        weight: 5
---

# Flash Messages

有时需要在同一用户的请求之间临时存储数据，例如表单提交后的错误或成功消息。Iris 会话包支持 flash 消息。

正如我们在[Sessions](/docs/base/sessions)一章中看到的，您可以这样初始化 session：

```go
import "github.com/kataras/iris/v12/sessions"

sess := sessions.New(sessions.Config{Cookie: "cookieName", ...})
```

```go
// [app := iris.New...]

app.Get("/path", func(ctx iris.Context) {
    session := sess.Start(ctx)
    // [...]
})
```

`Session` 包含以下存储、检索和删除 flash 消息的方法。

```go
SetFlash(key string, value interface{})
HasFlash() bool
GetFlashes() map[string]interface{}

PeekFlash(key string) interface{}
GetFlash(key string) interface{}
GetFlashString(key string) string
GetFlashStringDefault(key string, defaultValue string) string

DeleteFlash(key string)
ClearFlashes()
```

方法名就很浅显易懂。例如，如果您需要获取一条消息并在下一个请求时将其删除，请使用 `GetFlash`。当您只需要检索但不删除时，请使用 `PeekFlash`。

> 闪存消息不存储在数据库中。
