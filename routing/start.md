---
title: 路由入门
menu:
    docs:
        parent: 路由
        weight: 200
---

## The Handler type

A Handler, as the name implies, handle requests.

A Handler responds to an HTTP request.
It writes reply headers and data to the
Context.ResponseWriter() and then return.
Returning signals that the request is finished;
it is not valid to use the Context after or
concurrently with the completion of the Handler call.

Depending on the HTTP client software, HTTP protocol version,
and any intermediaries between the client and the iris server,
it may not be possible to read from the
Context.Request().Body after writing to the context.ResponseWriter().
Cautious handlers should read the Context.Request().Body first, and then reply.

Except for reading the body, handlers should not modify the provided Context.

If Handler panics, the server (the caller of Handler) assumes that
the effect of the panic was isolated to the active request.
It recovers the panic, logs a stack trace to the server error log
and hangs up the connection.

```go
type Handler func(iris.Context)
```

Once the handler is registered, we can use the returned [`Route`](https://godoc.org/github.com/kataras/iris/core/router#Route) instance to give a name to the handler registration for easier debugging or match relative paths in views. For more information, checkout the [[Reverse lookups|Routing-reverse-lookups]] section.

## Behavior

Iris' default behavior is to accept and register routes with paths like `/api/user`, without a trailing slash. If a client tries to reach `$your_host/api/user/` then the Iris router will automatically permant redirect this to `$your_host/api/user` in order to be handled by the registered route. This is the modern way to design APIs.

However, if you want to **disable path correction** for the requested resources you can pass the `iris.WithoutPathCorrection` option of the iris [[Configuration]] to your `app.Run`. Example:

```go
// [app := iris.New...]
// [...]

app.Listen(":8080", iris.WithoutPathCorrection)
```

If you want to keep the same handler and route for `/api/user` and `/api/user/` paths **without redirection**(common scenario) use just the `iris.WithoutPathCorrectionRedirection` option instead:

```go
app.Listen(":8080", iris.WithoutPathCorrectionRedirection)
```

## API

All HTTP methods are supported, developers can also register handlers on the same path with different methods.

The first parameter is the HTTP Method,
second parameter is the request path of the route,
third variadic parameter should contain one or more `iris.Handler` executed
by the registered order when a client requests for that specific resouce path from the server.

Example code:

```go
app := iris.New()

app.Handle("GET", "/contact", func(ctx iris.Context) {
    ctx.HTML("<h1> Hello from /contact </h1>")
})
```

In order to make things easier for the end-developer, iris provides method helpers for all HTTP Methods.
The first parameter is the request path of the route,
second variadic parameter should contains one or more iris.Handler executed
by the registered order when a user requests for that specific resouce path from the server.

Example code:

```go
app := iris.New()

// Method: "GET"
app.Get("/", handler)

// Method: "POST"
app.Post("/", handler)

// Method: "PUT"
app.Put("/", handler)

// Method: "DELETE"
app.Delete("/", handler)

// Method: "OPTIONS"
app.Options("/", handler)

// Method: "TRACE"
app.Trace("/", handler)

// Method: "CONNECT"
app.Connect("/", handler)

// Method: "HEAD"
app.Head("/", handler)

// Method: "PATCH"
app.Patch("/", handler)

// register the route for all HTTP Methods
app.Any("/", handler)

func handler(ctx iris.Context){
    ctx.Writef("Hello from method: %s and path: %s\n", ctx.Method(), ctx.Path())
}
```

### Offline Routes

There is one special method in Iris that you can use too. It's called `None` and you can use it to hide a route from outsiders but still able to call it from other route's handlers through the `Context.Exec` method. Each API Handle method returns the Route value back. A Route the `IsOnline` method which reports back the current state of that route. You can change the **state** of the route from **offline** to **online** and visa-versa through its `Route.Method` field's value. Of course each change of the router at serve-time requires an `app.RefreshRouter()` call which is safe to use. Take a look below a more complete example:

```go
// file: main.go
package main

import (
    "github.com/kataras/iris/v12"
)

func main() {
    app := iris.New()

    none := app.None("/invisible/{username}", func(ctx iris.Context) {
        ctx.Writef("Hello %s with method: %s", ctx.Params().Get("username"), ctx.Method())

        if from := ctx.Values().GetString("from"); from != "" {
            ctx.Writef("\nI see that you're coming from %s", from)
        }
    })

    app.Get("/change", func(ctx iris.Context) {

        if none.IsOnline() {
            none.Method = iris.MethodNone
        } else {
            none.Method = iris.MethodGet
        }

        // refresh re-builds the router at serve-time in order to
        // be notified for its new routes.
        app.RefreshRouter()
    })

    app.Get("/execute", func(ctx iris.Context) {
        if !none.IsOnline() {
            ctx.Values().Set("from", "/execute with offline access")
            ctx.Exec("NONE", "/invisible/iris")
            return
        }

        // same as navigating to "http://localhost:8080/invisible/iris"
        // when /change has being invoked and route state changed
        // from "offline" to "online"
        ctx.Values().Set("from", "/execute")
        // values and session can be
        // shared when calling Exec from a "foreign" context.
        // 	ctx.Exec("NONE", "/invisible/iris")
        // or after "/change":
        ctx.Exec("GET", "/invisible/iris")
    })

    app.Listen(":8080")
}
```

**How to run**

1. `go run main.go`
2. Open a browser at `http://localhost:8080/invisible/iris` and you'll see that you get a `404 not found` error,
3. however the `http://localhost:8080/execute` will be able to execute that route.
4. Now, if you navigate to the `http://localhost:8080/change` and refresh the `/invisible/iris` tab you'll see that you can see it.

## Grouping Routes

A set of routes that are being groupped by path prefix can (optionally) share the same middleware handlers and template layout.
A group can have a nested group too.

`.Party` is being used to group routes, developers can declare an unlimited number of (nested) groups.

Example code:

```go
app := iris.New()

users := app.Party("/users", myAuthMiddlewareHandler)

// http://localhost:8080/users/42/profile
users.Get("/{id:uint64}/profile", userProfileHandler)
// http://localhost:8080/users/messages/1
users.Get("/messages/{id:uint64}", userMessageHandler)
```

The same could be also written using the `PartyFunc` method which accepts the child router(the Party).

```go
app := iris.New()

app.PartyFunc("/users", func(users iris.Party) {
    users.Use(myAuthMiddlewareHandler)

    // http://localhost:8080/users/42/profile
    users.Get("/{id:uint64}/profile", userProfileHandler)
    // http://localhost:8080/users/messages/1
    users.Get("/messages/{id:uint64}", userMessageHandler)
})
```

## Path Parameters

Unlike other routers you'd seen, the Iris' one can handle various route paths without confliction between them.

Matches only GET "/".

```go
app.Get("/", indexHandler)
```

Matches all GET requests prefixed with `"/assets/**/*"`, it's a wildcard with `ctx.Params().Get("asset")` equals to any following path after the `/assets/`.

```go
app.Get("/assets/{asset:path}", assetsWildcardHandler)
```

Matches all GET requests prefixed with `"/profile/"`
and followed by a single path part.

```go
app.Get("/profile/{username:string}", userHandler)
```

Matches only GET `"/profile/me"` and
it does not conflict with `/profile/{username:string}`
or any root wildcard `/{root:path}`.

```go
app.Get("/profile/me", userHandler)
```


Matches all GET requests prefixed with `/users/`
and followed by a number which should be equal or higher than 1.

```go
app.Get("/user/{userid:int min(1)}", getUserHandler)
```

Matches all DELETE requests prefixed with `/users/`
and following by a number which should be equal or higher than 1.

```go
app.Delete("/user/{userid:int min(1)}", deleteUserHandler)
```

Matches all GET requests except the ones that are already handled by other routes. For example in this case by the above routes; `/`, `/assets/{asset:path}`, `/profile/{username}`,  `"/profile/me"`, `/user/{userid:int ...}`. It does not conflict with the rest of the routes(!).

```go
app.Get("{root:path}", rootWildcardHandler)
```

Matches all GET requests of:

1. /u/abcd maps to :alphabetical (if :alphabetical registered otherwise :string)
2. /u/42 maps to :uint (if :uint registered otherwise :int)
3. /u/-1 maps to :int (if :int registered otherwise :string)
4. /u/abcd123 maps to :string

```go
app.Get("/u/{username:string}", func(ctx iris.Context) {
	ctx.Writef("username (string): %s", ctx.Params().Get("username"))
})

app.Get("/u/{id:int}", func(ctx iris.Context) {
	ctx.Writef("id (int): %d", ctx.Params().GetIntDefault("id", 0))
})

app.Get("/u/{uid:uint}", func(ctx iris.Context) {
	ctx.Writef("uid (uint): %d", ctx.Params().GetUintDefault("uid", 0))
})

app.Get("/u/{firstname:alphabetical}", func(ctx iris.Context) {
	ctx.Writef("firstname (alphabetical): %s", ctx.Params().Get("firstname"))
})
```

Matches all GET requests of `/abctenchars.xml` and `/abcdtenchars` respectfully.

```go
app.Get("/{alias:string regexp(^[a-z0-9]{1,10}\\.xml$)}", PanoXML)
app.Get("/{alias:string regexp(^[a-z0-9]{1,10}$)}", Tour)
```

You may wonder what the `{id:uint64}` or `:path` or `min(1)` are. They are (typed) dynamic path parameters and functions can be registered on them. Learn more by reading the [[Path Parameter Types|Routing-path-parameter-types]].
