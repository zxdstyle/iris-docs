---
title: HTTP 缓存
menu:
    docs:
        parent: HTTP
        weight: 4
---

有时缓存静态内容的路由是很重要的，这样可以使您的 web 应用执行得更快，而不必花时间为客户端重新构建响应。

有两种方法可以实现 HTTP 缓存。一种是将每个处理程序的内容存储在服务器端，另一种是通过检查一些请求头并发送 `304 not modified`，以便让浏览器或任何兼容的客户端自行处理缓存。

Iris 通过 [iris/cache](https://github.com/kataras/iris/tree/master/cache) 提供服务器和客户端缓存, 提供了 `iris.Cache` and `iris.Cache304` 中间件。

让我们看看他们的用法。

### Cache

`Cache` 是一个中间件，为下一个处理程序提供服务器端缓存功能，可以这样使用：`app.Get("/", iris.Cache, aboutHandler)`。

`Cache` 只接受一个参数：缓存过期时间。如果 expiration 是无效值，<= 2秒，则 expiration 由 `cache control's maxage` 头执行

所有类型的响应都可以被缓存，模板、json、文本等等。

将其用于服务器端缓存，请参阅 `Cache304` 以获得更适合您的方法。

```go
func Cache(expiration time.Duration) Handler
```

需要更多选项和自定义的话，请使用 [kataras/iris/cache.Cache](https://godoc.org/github.com/kataras/iris/cache#Cache) ，它返回一个 [可以从中添加或删除 “Rules” 的结构体](https://godoc.org/github.com/kataras/iris/cache/client#Handler)。

### NoCache

`NoCache` 是一个中间件，它覆盖了Cache Control、Pragma 和Expires头，以便在浏览器的 back 和 forward 功能期间禁用缓存。

这个中间件的一个很好的用途是在HTML路由上；甚至在浏览器的“后退”和“前进”箭头按钮上刷新页面。

```go
func NoCache(Context)
```

有关相反的行为，请参见`StaticCache`。

### StaticCache

`StaticCache` 通过向客户端发送 “Cache Control” 和 “Expires” 头，返回用于缓存静态文件的中间件。它接受单个输入参数 “cacheDur” ，即持续时间，用来计算过期时间。

如果 `cacheDur <=0 `，则返回 “NoCache” 中间件，以禁用浏览器“后退”和“前进”操作之间的缓存。

使用方法: `app.Use(iris.StaticCache(24 * time.Hour))` 或 `app.Use(iris.StaticCache(-1))`.

```go
func StaticCache(cacheDur time.Duration)Handler
```

中间件是一个简单的处理程序，也可以在另一个处理程序中调用，例如：
```go
 cacheMiddleware := iris.StaticCache(...)

 func(ctx iris.Context){
  cacheMiddleware(ctx)
  [...]
}
```

### Cache304

每当 “If-Modified-Since” 请求头（时间）在 `time.Now() + expiresEvery` （始终与UTC值比较） 之前，`Cache304` 就会返回 `StatusNotModified` (304) 响应。

与 HTTP RCF 兼容的客户端（所有浏览器和 postman 之类的工具）将正确处理缓存。

使用该方法而不是服务器端缓存的唯一缺点是，该方法将发送 304 状态码而不是 200，因此，如果您将其与其他 micro 服务并排使用，则必须检查该状态码是否存在有效响应。

开发人员可以自由扩展此方法的具体逻辑，比如通过监听系统目录的手动更改并使用 `ctx.WriteWithExpiration` 方法基于文件修改日期的 `modtime`， 类似于`HandleDir`（发送状态为 OK（200）和浏览器磁盘缓存而不是 304）。

```go
func Cache304(expiresEvery time.Duration) Handler
```

示例: [https://github.com/kataras/iris/tree/master/_examples/response-writer/cache](https://github.com/kataras/iris/tree/master/_examples/response-writer/cache).
