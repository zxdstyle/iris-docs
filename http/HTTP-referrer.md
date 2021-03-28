---
title: HTTP Referrer
menu:
    docs:
        parent: HTTP
        weight: 2
---

Referer Policy HTTP 头控制请求中应该包含多少 Referer 信息（通过 Referer 请求头发送）。
The Referrer-Policy HTTP header controls how much referrer information (sent via the Referer header) should be included with requests.

详情参考 [developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy)

--------

Iris 使用 [Shopify's goreferrer](https://github.com/Shopify/goreferrer/pull/27) 扩展包暴露 `Context.GetReferrer()` 方法。

`GetReferrer` 方法从`Referer`（或`Referer`）头和 url 查询参数中提取并返回信息，如 [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/referer-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/referer-Policy) 中所述。


```go
GetReferrer() Referrer
```

`Referrer` 结构体如下:

```go
type (
    Referrer struct {
        Type       ReferrerType
        Label      string
        URL        string
        Subdomain  string
        Domain     string
        Tld        string         
        Path       string              
        Query      string                 
        GoogleType ReferrerGoogleSearchType
    }
```

“refererType” 是 `Referrer.Type` 的枚举值（indirect, direct, email, search, social）。可用类型包括：

```go
ReferrerInvalid
ReferrerIndirect
ReferrerDirect
ReferrerEmail
ReferrerSearch
ReferrerSocial
```

`GoogleType` 可用类型包括：

```go
ReferrerNotGoogleSearch
ReferrerGoogleOrganicSearch
ReferrerGoogleAdwords
```

## 示例

```go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.New()

    app.Get("/", func(ctx iris.Context) {
        r := ctx.GetReferrer()
        switch r.Type {
        case iris.ReferrerSearch:
            ctx.Writef("Search %s: %s\n", r.Label, r.Query)
            ctx.Writef("Google: %s\n", r.GoogleType)
        case iris.ReferrerSocial:
            ctx.Writef("Social %s\n", r.Label)
        case iris.ReferrerIndirect:
            ctx.Writef("Indirect: %s\n", r.URL)
        }
    })

    app.Listen(":8080")
}
```

`CURL` 示例:

```sh
curl http://localhost:8080?\
referrer=https://twitter.com/Xinterio/status/1023566830974251008

curl http://localhost:8080?\
referrer=https://www.google.com/search?q=Top+6+golang+web+frameworks\
&oq=Top+6+golang+web+frameworks
```
