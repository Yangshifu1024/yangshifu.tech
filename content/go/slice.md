---
title: "[翻译]Go Slices: usage and internals"
date: 2020-07-07T20:54:23+08:00
draft: false
summary: Go 切片：用法及内在本质
tags:
  - go
  - slice
---

> - 原文地址：[slices-intro](https://blog.golang.org/slices-intro)
>
> - 原作者：Andrew Gerrand
>
> - 原文发表于：2011-01-5
>
> - 所有图片均来源于原文

## 介绍
Go 语言的切片类型提供了方便且高效的方式来操作 __类型化的数据序列__。切片类型除了一些特殊的属性外，与其他程序语言的数组非常相似。本文将研究切片是什么以及如何使用它。

<!-- more -->

## 数组
切片类型是建立在数组之上的抽象概念，所以在深入切片之前我们首先需要了解数组。

数组的定义需要指定数组长度及元素类型。例如：`[4]int` 类型表示一个由 `4` 个整数组成的数组。数组的大小是固定的，长度是数组类型的一部分——`[4]int` 和 `[5]int` 是不同且不兼容的两个类型。可以使用常规的下标来索引数组，即 `s[n]` 访问数组 `s` 中从 `0` 开始的第 `n` 个元素。

```go
var a [4]int
a[0] = 1
i := a[0]
// i == 1
```

数组不需要显式的初始化；零值数组中的元素都已经被初始化为对应类型的零值，可以直接使用。
```go
// a[2] == 0, int 类型的零值
```

`[4]int` 在内存中表示为 __连续存储的 4 个整数__：

![Slice Array](slice-array.png)

Go 语言的数组是一个值。一个数组变量表示的是整个数组；而不是指向第一个元素的指针（比如 C 语言）。这就意味着当你对数组变量进行赋值或当作参数传递时，其接收方实际上是获得了该数组的一个 __副本__。（为了避免数组的复制，可以在传递时使用指向数组的指针，不过那就不是我们正在讨论的数组，而是 __数组的指针__ 了。）我们可以将数组看作一种包含索引而不是命名字段的结构体（struct）：一个固定大小的复合值。

数组可以有如下的字面量定义：

```go
b := [2]string{"Penn", "Teller"}
```

或者可以让编译器来决定数组长度：

```go
b := [...]string{"Penn", "Teller"}
```

无论以上哪种定义方式，`b` 都将是 `[2]string` 类型。

## 切片
数组有适用它们的地方，不过由于它们不够灵活，所以在 Go 的代码中并不常见。而切片却随处可见。切片基于数组，拥有更强的功能，也更便于使用。

切片的类型规范是 `[]T`，其中 `T` 代表切片元素的类型。与数组不同的是，切片类型没有指定长度。

切片的字面量定义除了长度之外与数组一致：

```go
letters := []string{"a", "b", "c", "d"}
```
切片还可以用内置的 `make` 函数来创建：

```go
func make([]T, len, cap) []T
```

其中 `T` 依然代表即将创建的切片元素类型。`make` 函数接受 3 个参数：__类型__、__长度（len）__、及 __可选的容量（cap）__。当使用 `make` 函数创建切片时，它在内部创建了一个数组，并返回了一个指向该数组的切片。

```go
var s []byte
s = make([]byte, 5, 5)
// s == []byte{0, 0, 0, 0, 0}
```

不提供容量（`cap`）参数时，其默认值与长度（`len`）一致：

```go
s := make([]byte, 5)
// s == []byte{0, 0, 0, 0, 0}
```

可以使用内置的 `len` 和 `cap` 函数来查看切片的长度与容量：

```go
// len(s) == 5
// cap(s) == 5
```

接下来的两个部分将讨论 __长度__ 与 __容量__ 之间的关系。

切片类型的零值是 `nil`。`len` 和 `cap` 函数在零值切片上都将返回 `0`。

切片可以由已经声明的切片或数组分割而来。两个索引加上一个分号（`:`）即可实现分割。比如：表达式 `b[1:4]` 创建了一个包含 `b` 中 `1` 到 `3` 三个元素的切片（在分割后的切片中下标为 `0` 到 `2`）。

```go
b := []byte{'g', 'o', 'l', 'a', 'n', 'g'}
// b[1:4] == []byte{'o', 'l', 'a'}, 其底层数组与 b 一致
```

在进行切片分割的表达式中，索引可以省略。在省略时，开始的索引默认值为 `0`，结束的索引默认值为 __切片的长度（len）__ ：

```go
b := []byte{'g', 'o', 'l', 'a', 'n', 'g'}
// b[:2] == []byte{'g', 'o'}
// b[2:] == []byte{'l', 'a', 'n', 'g'}
// b[:] == b
// 以下为译者注：
// b[:2] == b[0:2]
// b[2:] == b[2:len(b)]
// b[:] == b[0:len(b)] == b
```

还可以使用一个数组来创建切片：
```go
x := [3]string{"Лайка", "Белка", "Стрелка"}
s := x[:] // s 切片的底层为 x 数组
```

## 切片的本质
切片实际描述的是一个数组的片段，它由 `3` 部分组成：指向数组的指针、数组片段的长度以及容量（数组片段的最大长度）。

![Slice Struct](slice-struct.png)

我们之前通过 `make([]byte, 5)` 表达式创建的切片，其结构表示如下：

![Slice Struct](slice-1.png)

长度是切片引用的元素个数。容量是切片底层数组的元素个数（从切片指针指向的元素开始）。接下来的几个例子将阐述长度和容量的不同之处。

我们将一边分割切片 `s`，一边观察其内部的数据结构及它们和底层数组之间的关联：

```go
s = s[2:4]
```

![Slice Struct](slice-2.png)

分割过程并未复制切片的数据。只是创建了一个新的切片指向了原有的数组。这使得分割操作就如同数组下标操作一样高效。因此，修改新切片中的数据会影响旧切片。

```go
d := []byte{'r', 'o', 'a', 'd'}
e := d[2:]
// e == []byte{'a', 'd'}
e[1] = 'm'
// e == []byte{'a', 'm'}
// d == []byte{'r', 'o', 'a', 'm'}
```

之前的分割操作都没有使切片长度大于它的容量。我们也可以通过分割操作扩容 `s`：

```go
s = s[:cap(s)]
```

![Slice Struct](slice-3.png)

切片扩容时长度不能超过其容量。尝试这样做的时候将会引发运行时崩溃（`runtime panic`），和越界访问一个切片或数组一样。同样的，切片的长度也不能被分割到小于 `0`。

## 切片的扩容（`copy` 和 `append` 函数）
切片的扩容实际上是创建一个新的更大的切片，然后将数据拷贝到其中。这种技术就是其他程序语言中的动态数据的实际实现方式。下边的例子将创建一个两倍大小于切片 `s` 的切片 `t` 、从 `s` 拷贝内容到 `t`，最后将 `t` 的值赋值给 `s`：

```go
t := make([]byte, len(s), (cap(s)+1)*2) // +1 in case cap(s) == 0
for i := range s {
        t[i] = s[i]
}
s = t
```

循环部分的代码可以更简单地由内置函数 `copy` 来替代。如同它的名字一样，`copy` 函数复制源切片的数据到目标切片，然后返回复制的元素个数。

```go
func copy(dst, src []T) int
```

`copy` 函数支持复制不同长度的切片（复制的元素个数取决于两个切片中长度最小的那个）。此外，`copy` 函数可以正确处理源切片和目标切片为同一个底层数组，即切片重叠的情况。

使用 `copy` 函数，上边的代码可以简化为：

```go
t := make([]byte, len(s), (cap(s)+1)*2)
copy(t, s)
s = t
```

另一个切片的常见操作是向其尾部追加元素。下面的函数将 `byte` 类型的元素追加到一个 `[]byte` 切片，有必要还会进行扩容，并返回追加元素后的切片：

```go
func AppendByte(slice []byte, data ...byte) []byte {
    m := len(slice)
    n := m + len(data)
    if n > cap(slice) { // 如果有必要则进行扩容
        // 双倍长度扩容
        newSlice := make([]byte, (n+1)*2)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:n]
    copy(slice[m:n], data)
    return slice
}
```

然后我们可以这样使用 `AppendByte` 函数：

```go
p := []byte{2, 3, 5}
p = AppendByte(p, 7, 11, 13)
// p == []byte{2, 3, 5, 7, 11, 13}
```

类似 `AppendByte` 的一些工具函数非常有用，因为它完整地处理了追加元素时切片的扩容操作。它可以根据程序的特点来决定扩容的大小，或设定扩容的阈值。

但是大多数的程序不需要如此完整的控制，所以 Go 语言提供了一个内置的 `append` 函数来在大多数情况下直接使用，下面是它的方法签名：

```go
func append(s []T, x ...T) []T
```

`append` 函数追加元素到切片的尾部，并在合适的时候对切片进行扩容。

```go
a := make([]int, 1)
// a == []int{0}
a = append(a, 1, 2, 3)
// a == []int{0, 1, 2, 3}
```

追加一个切片到另一个切片时，使用 `...` 来展开第二个参数：

```go
a := []string{"John", "Paul"}
b := []string{"George", "Ringo", "Pete"}
a = append(a, b...) // 等同于 "append(a, b[0], b[1], b[2])"
// a == []string{"John", "Paul", "George", "Ringo", "Pete"}
```

零值切片可以当作 `0` 长度的切片来使用，可以定义一个切片零值，并在循环中向其追加元素：

```go
// Filter 返回一个只包含满足 fn 函数的元素的新切片
func Filter(s []int, fn func(int) bool) []int {
    var p []int // == nil
    for _, v := range s {
        if fn(v) {
            p = append(p, v)
        }
    }
    return p
}
```

## 潜在的陷阱
如同前面所说，在分割切片时并不会复制其底层的数组。整个数组会一直存在于内存中，直到它不再被引用。在偶然的情况下，这会导致程序保留整个数据在内存中，即使你只需要其中的一小部分。

如下所示，`FindDigits` 函数将一个文件加载到内存中，将其中第一个连续的数字当作一个新的切片 `[]byte` 返回到函数之外。

```go
var digitRegexp = regexp.MustCompile("[0-9]+")

func FindDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    return digitRegexp.Find(b)
}
```

这个函数确实只是简单地 `找到数字`，但是它返回的切片却指向了一个包含整个文件内容的数组。如此一来，只要返回的切片还在使用，垃圾回收器就无法对这个巨大的数据进行回收。几个有用的字节却让整个文件内容保留在了内存之中。

为了解决这个问题，我们可以只拷贝所需的数据到切片中，然后返回它：

```go
func CopyDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    b = digitRegexp.Find(b)
    c := make([]byte, len(b))
    copy(c, b)
    return c
}
```

可以用 `append` 函数来简化上面的函数，就当作一个练习留给读者。

<details>
    <summary>!! DONT CLICK ME!!</summary>
return append([]byte{}, b...)
</details>


## 延伸阅读

[Effective Go](https://golang.org/doc/effective_go.html) 包含了对 [slices](https://golang.org/doc/effective_go.html#slices) 和 [arrays](https://golang.org/doc/effective_go.html#arrays) 的更深入的探讨，并且 [Go 语言规范](https://golang.org/doc/go_spec.html)定义了 [slices](https://golang.org/doc/go_spec.html#Slice_types) 和 [相关的](https://golang.org/doc/go_spec.html#Length_and_capacity) [工具](https://golang.org/doc/go_spec.html#Making_slices_maps_and_channels) [函数](https://golang.org/doc/go_spec.html#Appending_and_copying_slices)。
