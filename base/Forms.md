---
title: 表单
menu:
    docs:
        parent: 基本功能
        weight: 6
---

可以使用以下上下文的方法检索表单、post数据和上传的文件。

```go
// FormValueDefault 根据 name 返回一个已解析的表单值，包括URL字段的查询参数和POST或PUT表单数据。
// 如果未找到，则返回 “def” 值。
FormValueDefault(name string, def string) string

// FormValue 根据 name 返回一个已解析的表单值， 包括URL字段的查询参数和POST或PUT表单数据。
FormValue(name string) string

// FormValues 返回所有已解析的表单数据，包括URL字段的查询参数以及POST或PUT表单数据。
//
// 默认表单的内存最大大小为32MB，可以通过以下方式更改，位于主要配置文件的 `iris.WithPostMaxMemory`，传递给`app.Run`的第二个参数。
//
// 注意: 有必要检查一下是否为 nil。
FormValues() map[string][]string

//  PostValueDefault 从 POST，PATCH或 PUT 的 body 中根据名称检索参数。
// 如果未找到，则返回 “def” 值。
PostValueDefault(name string, def string) string

// PostValue 从 POST，PATCH 或 PUT 的 body 中根据名称检索参数。
PostValue(name string) string

// PostValueTrim 从 POST，PATCH，或 PUT 的 body 中根据 name 检索，不带空格。
PostValueTrim(name string) string

// PostValueTrim 从 POST，PATCH，或 PUT 的 body 中根据 name 检索 int 类型的参数
// 如果未找到，则返回-1和非nil错误。
PostValueInt(name string) (int, error)

// PostValueIntDefault returns the parsed form data from POST, PATCH,
// or PUT body parameters based on a "name", as int.
//
// If not found returns or parse errors the "def".
PostValueIntDefault(name string, def int) int
// PostValueInt64 returns the parsed form data from POST, PATCH,
// or PUT body parameters based on a "name", as float64.
//
// If not found returns -1 and a no-nil error.
PostValueInt64(name string) (int64, error)
// PostValueInt64Default returns the parsed form data from POST, PATCH,
// or PUT body parameters based on a "name", as int64.
//
// If not found or parse errors returns the "def".
PostValueInt64Default(name string, def int64) int64
// PostValueInt64Default returns the parsed form data from POST, PATCH,
// or PUT body parameters based on a "name", as float64.
//
// If not found returns -1 and a non-nil error.
PostValueFloat64(name string) (float64, error)
// PostValueInt64Default returns the parsed form data from POST, PATCH,
// or PUT body parameters based on a "name", as float64.
//
// If not found or parse errors returns the "def".
PostValueFloat64Default(name string, def float64) float64
// PostValueInt64Default returns the parsed form data from POST, PATCH,
// or PUT body parameters based on a "name", as bool.
//
// If not found or value is false, then it returns false, otherwise true.
PostValueBool(name string) (bool, error)
// PostValues returns all the parsed form data from POST, PATCH,
// or PUT body parameters based on a "name" as a string slice.
//
// The default form's memory maximum size is 32MB, it can be changed by the
// `iris.WithPostMaxMemory` configurator at
// main configuration passed on `app.Run`'s second argument.
PostValues(name string) []string
// FormFile returns the first uploaded file that received from the client.
//
// The default form's memory maximum size is 32MB, it can be changed by the
//  `iris.WithPostMaxMemory` configurator at
// main configuration passed on `app.Run`'s second argument.
FormFile(key string) (multipart.File, *multipart.FileHeader, error)
```

## Multipart/Urlencoded Form

```go
func main() {
    app := iris.Default()

    app.Post("/form_post", func(ctx iris.Context) {
        message := ctx.FormValue("message")
        nick := ctx.FormValueDefault("nick", "anonymous")

        ctx.JSON(iris.Map{
            "status":  "posted",
            "message": message,
            "nick":    nick,
        })
    })

    app.Listen(":8080")
}
```

## Another example: query + post form

```sh
POST /post?id=1234&page=1 HTTP/1.1
Content-Type: application/x-www-form-urlencoded

name=manu&message=this_is_great
```

```go
func main() {
    app := iris.Default()

    app.Post("/post", func(ctx iris.Context) {
        id := ctx.URLParam("id")
        page := ctx.URLParamDefault("page", "0")
        name := ctx.FormValue("name")
        message := ctx.FormValue("message")
        // or `ctx.PostValue` for POST, PUT & PATCH-only HTTP Methods.

        app.Logger().Infof("id: %s; page: %s; name: %s; message: %s",
            id, page, name, message)
    })

    app.Listen(":8080")
}
```

```sh
id: 1234; page: 1; name: manu; message: this_is_great
```

## Upload files

Iris Context offers a helper for uploading files (saving files to host system's hard disk from request file data). Read more about the `Context.UploadFormFiles` method below.

UploadFormFiles uploads any received file(s) from the client
to the system physical location "destDirectory".

The second optional argument "before" gives caller the chance to
modify the *miltipart.FileHeader before saving to the disk,
it can be used to change a file's name based on the current request,
all FileHeader's options can be changed. You can ignore it if
you don't need to use this capability before saving a file to the disk.

Note that it doesn't check if request body streamed.

Returns the copied length as int64 and
a not nil error if at least one new file
can't be created due to the operating system's permissions or
`net/http.ErrMissingFile` if no file received.

If you want to receive & accept files and manage them manually you can use the `Context.FormFile`
instead and create a copy function that suits your needs, the below is for generic usage.

The default form's memory maximum size is 32MB, it can be changed by the `iris#WithPostMaxMemory` configurator at main configuration passed on `app.Run`'s second argument.

```go
UploadFormFiles(destDirectory string,
                before ...func(Context, *multipart.FileHeader)) (n int64, err error)
```

Example Code:

```go
const maxSize = 5 << 20 // 5MB

func main() {
    app := iris.Default()
    app.Post("/upload", iris.LimitRequestBodySize(maxSize), func(ctx iris.Context) {
        //
        // UploadFormFiles
        // uploads any number of incoming files ("multiple" property on the form input).
        //

        // The second, optional, argument
        // can be used to change a file's name based on the request,
        // at this example we will showcase how to use it
        // by prefixing the uploaded file with the current user's ip.
        ctx.UploadFormFiles("./uploads", beforeSave)
    })

    app.Listen(":8080")
}

func beforeSave(ctx iris.Context, file *multipart.FileHeader) {
    ip := ctx.RemoteAddr()
    // make sure you format the ip in a way
    // that can be used for a file name (simple case):
    ip = strings.Replace(ip, ".", "_", -1)
    ip = strings.Replace(ip, ":", "_", -1)

    // you can use the time.Now, to prefix or suffix the files
    // based on the current time as well, as an exercise.
    // i.e unixTime :=	time.Now().Unix()
    // prefix the Filename with the $IP-
    // no need for more actions, internal uploader will use this
    // name to save the file into the "./uploads" folder.
    file.Filename = ip + "-" + file.Filename
}
```

How to `curl`:

```sh
curl -X POST http://localhost:8080/upload \
  -F "files[]=@./myfile.zip" \
  -F "files[]=@./mysecondfile.zip" \
  -H "Content-Type: multipart/form-data"
```

More examples can be found at <https://github.com/kataras/iris/tree/master/_examples/request-body> and <https://github.com/kataras/iris/tree/master/_examples/file-server>.