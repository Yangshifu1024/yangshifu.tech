---
title: "Stringer"
date: 2020-07-03T16:30:42+08:00
summary: "stringer 是一个 Go 代码生成工具"
draft: false
tags:
  - go
  - cmd
---

## 需求

在 Go 中，我们一般用自定义类型来实现枚举的功能，如下代码所示：

```go
// sport.go
type Sport int

const (
	Football Sport = 1 // 足球
	Basketball Sport = 2 // 篮球
	TableTennis Sport = 3 // 乒乓球
)
```

而在实际的使用过程中，我们还需要枚举值能够自解释，比如获取该枚举值定义的实际意义——注释。

<!-- more -->

## 解决

Go 的 `x/tools` 包下提供了很多有用的命令，其中的 `stringer` 可以帮助我们实现上面的需求。

使用以下命令安装：

```shell
$ go get -u golang.org/x/tools/cmd/stringer
```

执行
```shell
$ stringer -type=Sport --linecomment
```

即可在 `sport.go` 文件所在目录生成 `sport_string.go`：

![Sport String](sport_string_go.png)

接下来可以这样使用：
```go
func main() {
	fmt.Println(Football.String())
}
```
```shell
$ go run *.go
足球
```

由此，我们可以在使用常量枚举时获取其真实意义，在 `API` 返回和打印日志时都非常有用。

## 优化

假设我们大量用到了 `stringer` 来生成代码，那么针对每个类型都执行一次 `stringer` 命令是不可接受的。比如我们新增了 `people.go`：

```go
// people.go
type People int

const (
	Bob  People = 1 // 鲍勃
	Lily People = 2 // 莉莉
)
```

这里我们使用 `go generate` 命令来简化操作。

改写 `sport.go` 如下：

```go
// sport.go

//go:generate stringer -type=Sport --linecomment
type Sport int

const (
	Football Sport = 1 // 足球
	Basketball Sport = 2 // 篮球
	TableTennis Sport = 3 // 乒乓球
)
```

改写 `people.go` 如下：
```go
// people.go

//go:generate stringer -type=People --linecomment
type People int

const (
	Bob  People = 1 // 鲍勃
	Lily People = 2 // 莉莉
)
```

修改之后，我们只需要在项目根目录执行 `go generate` 即可得到直接执行 `stringer -type Sport --linecomment` 的效果。

## 代码

本文示例代码

[https://github.com/MrHeaaaavy/stringer_example](https://github.com/MrHeaaaavy/stringer_example)
