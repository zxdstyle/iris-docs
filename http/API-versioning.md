---
title: API 版本控制
menu:
    docs:
        parent: HTTP
        weight: 3
---

[版本控制](https://github.com/kataras/iris/tree/master/versioning) 子包提供了 [semver](https://semver.org/) API的版本控制功能。它实现了 [api指南](https://github.com/byrondover/api guidelines/blob/master/guidelines.md#版本控制) 中的所有建议还要多。

版本比较由 [go version](https://github.com/hashicorp/go-version) 处理。它支持`>=1.0，<3` 等模式进行匹配。


```go
import (
    // [...]

    "github.com/kataras/iris/v12"
    "github.com/kataras/iris/v12/versioning"
)
```

## 特点

- 支持单个路由进行版本匹配，一个普通的iris处理程序可以通过 `version => handler` 构成的 Map 实现类似 「switch」 语法的功能。
- 支持路由组版本化和弃用API
- 支持版本匹配，如 `>=1.0，<2.0` 或 `2.0.1` 等。
- 找不到版本异常处理（只需在 Map 中 添加 customNotMatchVersionHandler即可）
- 从 `Accept` 和 `Accept version` 请求头中检索版本（也可以通过中间件定制）
- 如果找到匹配版本，用 `X-API-Version` 响应头响应。
- 通过 `Deprecated` 包装器提供可自定义的 `X-API-Warn` 、`X-API-deprecation-Date`、`X-API-deprecation-Info` 响应头的弃用信息设置。

## 获取版本

当前请求版本可以通过 `versioning.GetVersion(ctx)` 获取。

 `GetVersion` 默认会从以下两个方式获取版本:
- `Accept` 请求头, 例如 `Accept: "application/json; version=1.0"`
- `Accept-Version` 请求头, 例如 `Accept-Version: "1.0"`

您还可以通过中间件为处理程序设置自定义版，使用上下文存储值即可。
例如：

```go
func(ctx iris.Context) {
    ctx.Values().Set(versioning.Key, ctx.URLParamDefault("version", "1.0"))
    ctx.Next()
}
```

## 匹配相应版本的 Hanlder

`versioning.NewMatcher(versioning.Map) iris.Handler` 可以创建单个处理程序，然后由他来决定请求的版本该使用哪个处理程序。

```go
app := iris.New()

// 所有版本共用的中间件
myMiddleware := func(ctx iris.Context) {
    // [...]
    ctx.Next()
}

myCustomNotVersionFound := func(ctx iris.Context) {
    ctx.StatusCode(iris.StatusNotFound)
    ctx.Writef("%s version not found", versioning.GetVersion(ctx))
}

userAPI := app.Party("/api/user")
userAPI.Get("/", myMiddleware, versioning.NewMatcher(versioning.Map{
    "1.0":               sendHandler(v10Response),
    ">= 2, < 3":         sendHandler(v2Response),
    versioning.NotFound: myCustomNotVersionFound,
}))
```

### 弃用

使用 `versioning.Deprecated(handler iris.Handler, options versioning.DeprecationOptions) iris.Handler` 可以标记指定 Hanlder 版本将被弃用。


```go
v10Handler := versioning.Deprecated(sendHandler(v10Response), versioning.DeprecationOptions{
    // 默认提示: "WARNING! You are using a deprecated version of this API."
    WarnMessage string 
    DeprecationDate time.Time
    DeprecationInfo string
})

userAPI.Get("/", versioning.NewMatcher(versioning.Map{
    "1.0": v10Handler,
    // [...]
}))
```

这将使处理程序将这些响应头发送到客户端：

- `"X-API-Warn": options.WarnMessage`
- `"X-API-Deprecation-Date": context.FormatTime(ctx, options.DeprecationDate))`
- `"X-API-Deprecation-Info": options.DeprecationInfo`

> 如果不需要弃用时间和信息则可以使用 versioning.DefaultDeprecationOptions

## 路由组版本控制

也可以对路由组进行版本控制。

使用 `versioning.NewGroup(version string) *versioning.Group` 方法既可以创建一个拥有版本控制功能的路由组。
`versioning.RegisterGroups(r iris.Party, versionNotFoundHandler iris.Handler, groups ...*versioning.Group)` 必须在结尾处调用才能将路由注册到指定 `Party`.

```go
app := iris.New()

userAPI := app.Party("/api/user")
// [... 静态版本，中间件等可以放在这].

userAPIV10 := versioning.NewGroup("1.0")
userAPIV10.Get("/", sendHandler(v10Response))

userAPIV2 := versioning.NewGroup(">= 2, < 3")
userAPIV2.Get("/", sendHandler(v2Response))
userAPIV2.Post("/", sendHandler(v2Response))
userAPIV2.Put("/other", sendHandler(v2Response))

versioning.RegisterGroups(userAPI, versioning.NotFoundHandler, userAPIV10, userAPIV2)
```

> 中间件只能注册到实际的 `iris.Party` 中，使用我们上面学过的方法，例如 `versioning.Match` 以便在请求 `x` 或没有版本时检测要执行的 code/handler。

### 路由组弃用

只需在即将弃用的接口上调用即可 `Deprecated(versioning.DeprecationOptions)` 在要通知API使用者。

```go
userAPIV10 := versioning.NewGroup("1.0").Deprecated(versioning.DefaultDeprecationOptions)
```

## Hanlder 内手动比较版本

```go
// 检查 「version」 和 「is」 是否匹配， 「is」 可以像这样书写：>= 1, < 3
If(version string, is string) bool
```

```go
// 与 「If」 方法类似，但是需要一个上下文来读取请求的版本。
Match(ctx iris.Context, expectedVersion string) bool
```

```go
app.Get("/api/user", func(ctx iris.Context) {
    if versioning.Match(ctx, ">= 2.2.3") {
        // [logic for >= 2.2.3 version of your handler goes here]
        return
    }
})
```
