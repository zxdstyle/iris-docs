---
title: Cookies
menu:
    docs:
        parent: 基本功能
        weight: 2
---

Cookies 可以通过 Context 的请求实例访问。`ctx.Request()` 返回一个 `net/http#Request`。

不过，Iris 的 `Context` 提供了一些帮助程序，使您可以更容易地访问 cookie 的最常见用例，而无需任何定制的附加代码（如果您仅使用请求的cookie方法，则需要这些代码）。

## 设置 Cookie

`SetCookie` 和 `UpsertCookie` 方法添加或设置（必要时替换）cookie。

“options” 的最后一个参数是可选的。`CookieOption` 可用于修改 cookie 。稍后您将看到可用选项是什么，可以根据您的web应用程序需求添加自定义选项，这也有助于避免在代码中重复书写代码。

在我们继续之前，`Context` 有一个 `SetSameSite(http.SameSite)` 方法也是。它设置 ["SameSite"](https://web.dev/samesite-cookies-explained/) 所有要发送的cookie字段。

```go
SetCookie(cookie *http.Cookie, options ...CookieOption)
```

如果需要，还可以使用 `SetCookieKV` 方法，该方法不需要导入`net/http`包。

```go
SetCookieKV(name, value string, options ...CookieOption)
```

请注意，`SetCookieKV` 设置的 cookie 默认过期时间是 365 天。您可以使用 `CookieExpires` Cookie 选项（见下文），也可以全局设置 `kataras/iris/Context.SetCookieKVExpiration` 文件 包级别变量。

`CookieOption` 只支持 `func(*http.Cookie)` 类型。

**设置PATH**

```go
CookiePath(path string) CookieOption
```

**设置过期时间**

```go
iris.CookieExpires(durFromNow time.Duration) CookieOption
```

**HttpOnly**

```go
iris.CookieHTTPOnly(httpOnly bool) CookieOption
```

> HttpOnly 默认 true for `RemoveCookie` and `SetCookieKV`.

让我们进一步研究一下 cookie 编码。

**Encode**

在添加cookie时提供编码功能。

接受 `CookieEncoder` 并将cookie的值设置为编码值。

`SetCookie` 和 `SetCookieKV` 方法都可以接收。

```go
iris.CookieEncode(encode CookieEncoder) CookieOption
```

**Decode**

在检索cookie时提供解码功能。

接受 `CookieDecoder`，并在 `GetCookie` 返回之前将cookie的值设置为已解码的值。

```go
iris.CookieDecode(decode CookieDecoder) CookieOption
```

`CookieEncoder` 可以描述为:

CookieEncoder应该对cookie值进行编码。

* 应接受 cookie 的名称作为其第一个参数
* 第二个参数是cookie 的 ptr 值。
* 如果编码操作失败，则应返回编码值或空值。
* 如果编码操作失败，则应返回错误。

```go
CookieEncoder func(cookieName string, value interface{}) (string, error)
```

≈`CookieDecoder`:

CookieDecoder 可以解码 cookie 值。

* 应该接受cookie的名称作为第一个参数，
* 第二个参数是编码的cookie值，第三个参数是解码的ptr值。
* 如果解码操作失败，则应返回解码值或空值。
* 如果解码操作失败，则应返回错误。

```go
CookieDecoder func(cookieName string, cookieValue string, v interface{}) error
```

错误不会被打印出来，所以您必须知道自己在做什么，并且记住：如果您使用AES，它只支持16、24或 32 字节的密钥大小。

您要么需要提供准确的数量，要么从您键入的内容派生密钥。

## 获取 Cookie

“GetCookie” 根据名称检索 cookie 的值，如果未找到任何内容，则返回空字符串。

```go
GetCookie(name string, options ...CookieOption) string
```

如果您想要的值大于此值，请改用以下值：

```go
cookie, err := ctx.Request().Cookie("name")
```

## 获取所有 Cookies

那个 `ctx.Request().Cookies()` 方法返回所有可用请求Cookie的切片。有时您需要修改它们或对它们中的每一个执行操作，最简单的方法是使用 `VisitAllCookies` 方法。

```go
VisitAllCookies(visitor func(name string, value string))
```

## 删除 Cookie

`RemoveCookie` 删除指定名称和 path = "/" 的 cookie 值

提示: 可以通过 `iris.CookieCleanPath` 更改 cookie 的 PATH，例如 : `RemoveCookie("nname", iris.CookieCleanPath)`.

另外，请注意，默认行为是将其 `HttpOnly` 设置为 true。它根据 web 标准执行 cookie 的删除。

```go
RemoveCookie(name string, options ...CookieOption)
```

继续了解 [HTTP Sessions](/docs/base/sessions).
