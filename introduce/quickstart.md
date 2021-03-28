---
title: 快速开始
menu:
    docs:
        parent: 介绍
        weight: 3
---

创建一个空文件，假设其名称为 `example.go` ，然后打开它并复制粘贴下面的代码。

```go
package main

import "github.com/kataras/iris/v12"

func main() {
app := iris.Default()
app.Use(myMiddleware)

    app.Handle("GET", "/ping", func(ctx iris.Context) {
        ctx.JSON(iris.Map{"message": "pong"})
    })

    // 监听服务器 http://localhost:8080 即将到来的 HTTP 请求
    app.Listen(":8080")
}

func myMiddleware(ctx iris.Context) {
    ctx.Application().Logger().Infof("Runs before %s", ctx.Path())
    ctx.Next()
}
```

启动终端会话并执行以下操作。

```shell
# 运行 example.go 然后打开浏览器访问 http://localhost:8080/ping
$ go run example.go
```

## 更多细节

我们简单地看看启动和运行有多容易。

```go
package main

import "github.com/kataras/iris/v12"

func main() {
	app := iris.New()
	
	// 加载 "./views" 目录的所有 html 后缀名的模板文件，然后使用 `html/template` 标准扩展包渲染
	app.RegisterView(iris.HTML("./views", ".html"))

	// Method:    GET
	// Resource:  http://localhost:8080
	app.Get("/", func(ctx iris.Context) {
		
		// 传递 message 变量，值为 "Hello world!" 字符串
		ctx.ViewData("message", "Hello world!")
		
		// 渲染模板文件: ./views/hello.html
		ctx.View("hello.html")
	})

	// Method:    GET
	// Resource:  http://localhost:8080/user/42
	//
	// 需要自定义匹配规则?
	// 小事一桩;
	// 只需将参数的类型标记为 `string` 接受任何内容并利用 `regexp` 函数,例如：
	// app.Get("/user/{id:string regexp(^[0-9]+$)}")
	app.Get("/user/{id:uint64}", func(ctx iris.Context) {
		userID, _ := ctx.Params().GetUint64("id")
		ctx.Writef("User ID: %d", userID)
	})

	// 指定地址启动服务器。
	app.Listen(":8080")
}
```


```html
<!-- file: ./views/hello.html -->
<html>
<head>
    <title>Hello Page</title>
</head>
<body>
    <h1>{{.message}}</h1>
</body>
</html>
```

想在源代码更改时自动重启应用程序吗？安装 iris cli 工具并执行 `iris cli run` 而不是 `go run main.go`.

在下一节中，我们将了解有关路由的更多信息。