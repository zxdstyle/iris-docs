---
title: 安装
menu:
    docs:
        parent: 介绍
        weight: 2
---

Iris 是一个跨平台的软件。

唯一的要求是 Go 编程语言，版本 1.14 及以上。

```shell
$ go env -w GO111MODULE=on
// Install
$ go get github.com/kataras/iris/v12@v12.1.8
```

或者编辑您项目的 go.mod 文件.

```go
module your_project_name

go 1.14

require (
    github.com/kataras/iris/v12 v12.1.8
)
```

```shell
$ go build
```

故障排除

如果在安装过程中出现网络错误，请确保设置了有效的 GOPROXY 环境变量。

```shell
go env -w GOPROXY=https://goproxy.cn,https://gocenter.io,https://goproxy.io,direct
```
