---
title: 内容协商
menu:
    docs:
        parent: HTTP
        weight: 4
---

**有时，服务器应用程序需要在同一 URI 中为不同的资源提供服务**. 当然，这可以手动完成，手动检查 “Accept” 请求头并推送请求的内容表单。但是，由于您的应用程序管理更多的资源和不同类型的表示，这可能会非常痛苦，因为您可能需要检查“Accept Charset”、“Accept Encoding”、设置一些 服务器端优先级、正确处理错误等等。

Go中的一些web框架已经实现这样的功能，但它们做得不是很正确：

- 它们根本不处理 accept-charset
- 它们根本不处理 accept-encoding 
- 他们不发送错误状态码（406不可接受），因为 RFC 提议和更多...

但是，幸运的是，**Iris始终遵循最佳实践和 Web 标准**。

基于:
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Charset
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Encoding

实践:
- https://github.com/kataras/iris/pull/1316/commits/8ee0de51c593fe0483fbea38117c3c88e065f2ef#diff-15cce7299aae8810bcab9b0bf9a2fdb1

---------

## 示例

```go
type testdata struct {
	Name string `json:"name" xml:"Name"`
	Age  int    `json:"age" xml:"Age"`
}
```

使用 “gzip” 编码算法将资源呈现为 `application/json` 或 `text/xml` 或 `application/xml`

- 当客户端的 accept 头包含其中一个
- 或者JSON（第一个声明），如果accept为空，
- 当客户端的accept-encoding头包含“gzip”或为空时。

```go
app.Get("/resource", func(ctx iris.Context) {
	data := testdata{
		Name: "test name",
		Age:  26,
	}

        ctx.Negotiation().JSON().XML().EncodingGzip()

	_, err := ctx.Negotiate(data)
	if err != nil {
		ctx.Writef("%v", err)
	}
})

```

**或者**在中间件中定义它们，并在最终处理程序中调用 Negotiate 进行协商。

```go
ctx.Negotiation().JSON(data).XML(data).Any("content for */*")
ctx.Negotiate(nil)
```

```go
app.Get("/resource2", func(ctx iris.Context) {
	jsonAndXML := testdata{
		Name: "test name",
		Age:  26,
	}

	ctx.Negotiation().
		JSON(jsonAndXML).
		XML(jsonAndXML).
		HTML("<h1>Test Name</h1><h2>Age 26</h2>")

	ctx.Negotiate(nil)
})
```

[完整示例](https://github.com/kataras/iris/blob/master/_examples/response-writer/content-negotiation/main.go#L22).

## 文档
[Context.Negotiation](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3342) 方法创建并返回 negotiation builder，以便优先为特定的内容类型、字符集和编码算法构建服务器端可用的内容。

```go
Context.Negotiation() *context.NegotiationBuilder
```

[Context.Negotiate](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3402) 用于在同一URI上为资源的不同表示形式提供服务的方。没有匹配到 `mime type(s)` 时将返回 `context.ErrContentNotSupported`。

- "v" 可以说 [iris.N](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3298-L3309) 结构体.
- "v" 可以是实现 [context.ContentSelector](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3272) 接口的任何值.
- "v" 可以是实现 [context.ContentNegotiator](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3281) 接口的任何值.
- "v" 可以是结构体(JSON, JSONP, XML, YAML) 或者字符串(TEXT, HTML) 或者 []byte(Markdown, Binary) 或 []byte 任何匹配 mime 类型的字节.
- 如果 "v" 的值为 nil， `Context.Negotitation()` 将改为使用 builder 的内容，否则 "v" 会覆盖 builder 的内容（服务器 mime 类型仍由其注册、受支持的mime列表检索）


- 设置 mime 类型优先级 [Negotiation().MIME.Text.JSON.XML.HTML...](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3500-L3621).
- 设置字符集优先级 [Negotiation().Charset(...)](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3640).
- 设置编码算法优先级 [Negotiation().Encoding(...)](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3652-L3665).
- 修改接受者 [Negotiation().Accept./Override()/.XML().JSON().Charset(...).Encoding(...)...](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3774-L3877).

```go
Context.Negotiate(v interface{}) (int, error)
```