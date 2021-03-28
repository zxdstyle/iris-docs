---
title: HTTP 方法重写
menu:
    docs:
        parent: HTTP
        weight: 1
---

在开发和升级 REST API 时，使用特定的自定义 HTTP 头（如X-HTTP方法重写）是非常方便的。部署基于REST API的web服务时，**服务器**和**客户端**都可能遇到访问限制。

有些防火墙不支持PUT、DELETE 或 PATCH 请求。

-----------

[方法重写](https://github.com/kataras/iris/tree/master/middleware/methodoverride)包装器允许您在客户端不支持的地方使用 HTTP 动词，如PUT或DELETE。

### 服务端

```go
package main

import (
    "github.com/kataras/iris/v12"
    "github.com/kataras/iris/v12/middleware/methodoverride"
)

func main() {
    app := iris.New() 

    mo := methodoverride.New( 
        // 默认是 nil.
        methodoverride.SaveOriginalMethod("_originalMethod"), 
        // Default values. 
        // 
        // methodoverride.Methods(http.MethodPost), 
        // methodoverride.Headers("X-HTTP-Method",
        //                        "X-HTTP-Method-Override",
        //                        "X-Method-Override"), 
        // methodoverride.FormField("_method"), 
        // methodoverride.Query("_method"), 
    ) 
    // Register it with `WrapRouter`. 
    app.WrapRouter(mo)

    app.Post("/path", func(ctx iris.Context) {
        ctx.WriteString("post response")
    })

    app.Delete("/path", func(ctx iris.Context) {
        ctx.WriteString("delete response")
    })

    // [...app.Run]
}
```

### 客户端

```js
fetch("/path", {
    method: 'POST',
    headers: {
      "X-HTTP-Method": "DELETE"
    },
  })
  .then((resp)=>{
      // 响应体将是「delete」 响应. 
 })).catch((err)=> { console.error(err) })
```
