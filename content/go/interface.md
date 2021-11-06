---
title: "[翻译]Go Data Structures: Interfaces"
date: 2020-06-18T22:18:22+08:00
draft: false
summary: Go 数据结构：接口
cagetories:
- go
tags:
- go
- interface
---
> - 原文地址：[Go Data Structures: Interfaces](https://research.swtch.com/interfaces)
>
> - 原作者：Russ Cox
>
> - 原文发表于：2009-12-01
>
> - 所有图片均来源于原文
>
> - 译者注：
>    - 由于本文年代久远，部分内容可能已经过时，请读者自行甄别；
>    - 译者在能力范围之内，会在译文中进行标注过时内容以及新的实现；
>    - 文中部分链接已失效，译者已尽力恢复。

从语言设计的角度来看，对于我个人而言，Go 语言的接口是最令我兴奋的部分——在编译时是静态且进行类型检查的，而在运行时则是动态的。如果可以迁移一个语言特性到别的开发语言，那么必将是接口。

本文是我在实现 Go 语言编译器：`6g` `8g` `5g` （译者注：在 Go 1.5 实现自举之后已经不存在这些编译器，这些编译器存在于早期的 C 实现中。） 中接口值时的观点。与此同时， Ian Lance Taylor 写了两篇关于 `gccgo` 中接口值实现的文章（[1](http://www.airs.com/blog/archives/277) [2](http://www.airs.com/blog/archives/276)）。两种实现的方式几乎是相同的，最大的不同是：本文配了图。
在讨论具体实现之前，让我们先了解为何要实现它。

<!-- more -->

## 用法
Go 语言的接口可以让你像在其他动态语言，如 Python 语言一样使用 [Duck Typing(鸭子类型)](http://en.wikipedia.org/wiki/Duck_typing)，并且编译器还能辅助你捕获一些明显的错误，比如：在期望一个包含 `Read` 方法的值时却传递了一个 `int`，或者调用 `Read` 方法时参数个数错误。使用接口，首先就需要定义一个接口类型，比如下边的 `ReadCloser`：

```go
type ReadCloser interface {
    Read(b []byte) (n int, err os.Error)
    Close()
}
```

然后定义一个新的函数，其参数为 `ReadCloser` 类型。比如如下函数中，重复调用 `Read` 方法去获取所有的数据，然后调用 `Close` 进行关闭：

```go
func ReadAndClose(r ReadCloser, buf []byte) (n int, err os.Error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    r.Close()
    return
}
```

上述代码中的 `ReadAndClose` 函数，接收任意实现了 `Read` 和 `Close` 方法的类型，只要它的方法签名正确。与 Python 等其他语言不一样的是，如果你传递了错误的类型，那么编译时就会爆出错误，而不是在运行时。

但是接口也不仅限于静态检查，也可以动态地检查某个接口值是否包含指定的方法。比如：

```go
type Stringer interface {
    String() string
}

func ToString(any interface{}) string {
    if v, ok := any.(Stringer); ok {
        return v.String()
    }
    switch v := any.(type) {
    case int:
        return strconv.Itoa(v)
    case float:
        return strconv.Ftoa(v, 'g', -1)
    }
    return "???"
}
```

`any` 参数定义为 `interface{}`，这就意味着它不保证它其中包含什么方法，因为它可以是任何类型。在 `if` 语句中的 `comma ok` （译者注：在这里指的是上述代码中的 `v, ok := ...`） 赋值方式，要求 `any` 可以被转换为包含 `String()` 方法的 `Stringer` 类型。如果 `ok == true`，那么在接下来的语句中就直接调用 `String()` 方法并返回。此外，在函数结束之前，`switch` 语句对 `any` 的类型进行了条件选择。这个函数可以看作 [fmt 包](http://golang.org/pkg/fmt/) 的一个精简版本。（正常来说，上述代码中的 `if` 段可以被在 `switch` 条件中增加一个 `case Stringer` 来替换，这里展示了另一种值得注意的明确做法。）

一个简单的例子，为 64 位整数定义一个 `String` 方法来输出它的二进制值，定义一个 `Get` 方法来返回它的实际值：

```go
type Binary uint64

func (i Binary) String() string {
    return strconv.Uitob64(i.Get(), 2)
}

func (i Binary) Get() uint64 {
    return uint64(i)
}
```

`Binary` 类型可以被传递到 `ToString` 函数中，然后该函数使用 `Binary` 的 `String` 方法来格式化输出，即使程序员从来没有显式地声明 `Binary` 类型实现了 `Stringer` 接口。因为这是不需要的：Go 语言的运行时会看到 `Binary` 类型包含了 `String` 方法，所以它就实现了 `Stringer` 接口，即便 `Binary` 类型的作者甚至都没有听说过 `Stringer` 接口。

这些示例表明，即使在编译时检查所有隐式转换，显式接口间转换也可以在运行时查询方法集。[Effective Go](http://golang.org/doc/effective_go.html#interfaces) 中包含更多接口值的详细使用示例。

## 接口值 (Interface Values)
具有方法的语言通常有以下两种实现：一是准备一个方法表来静态调用（如 C++ 和 Java），二是在调用时进行方法查找（如 Smalltalk、Javascript 和 Python），然后缓存起来以提高调用效率。Go 语言则位于两种实现之间，它具有方法表，但在运行时对其计算。我不知道 Go 是否是第一个使用此技术的语言，但这种方式确实不常见。

**以下所有操作假定在 32 位机器上执行。**

在上面的例子中 `Binary` 类型是由两个 `32-bit` 字组成的 `64-bit` 整数：

{% asset_img gointer1.png%}

接口类型的值在内存中表现为两个字组成的一对数据，其中一个字为指向类型信息的指针，另一个则是指向关联值的指针。将 `b` 赋值给接口类型的值时会设置这两个指针：

{% asset_img gointer2.png%}

(上图中灰色的箭头强调说明这些指针是隐式的，在 Go 程序中不可见的。)

接口值中的第一个 `32-bit` 字指向了 `interface table` 或称为 `itable`（读音为 `i-table`；在[源码的 C 实现](https://github.com/golang/go/blob/weekly.2009-12-07/src/pkg/runtime/iface.c#L24)中，其名称为 `Itab`）。`itable` 的头部是一些关于类型的元数据，然后是指向函数的指针列表。注意这里的 `itable` 只对应 `interface` 类型，而不是动态的接口类型。就我们的例子而言，`Stringer` 的 `itable` 中仅包含了适配 `Binary` 类型中适配 `Stringer` 接口的 `String` 方法，`Binary` 的其他方法在此 `itable` 中不可见，如 `Get`。

接口值中的第二个 `32-bit` 字指向了实际的值，即 `b` 的副本。就像 `var c uint64 = b` 创建了 `b` 的一个副本一样，`var s Stringer = b` 同样也创建了一个 `b` 的副本：如果 `b` 发生了改变，那么 `s` 和 `c` 不会受到影响。存储在接口中的值可能为任意大小，但是只有一个 `32-bit` 字用于保存接口结构中的值，因此在内存分配时，堆上创建了一块内存，并将指针记录在这个 `32-bit` 字中。（接口值在某些时候会进行优化，后面会谈到。）

当在检查一个接口值是否匹配一个类型时，比如上述代码中的 `switch`，Go 编译器生成了类似 `s.tab->type` 这样的 C 代码来获取类型的指针并将其与期望的类型做比较。如果类型匹配，则接口值 `s.data` 就会被通过解引用进行拷贝。

为了实现 `s.String()` 方法的调用，Go 编译器生成了类似 `s.tab->fun[0](s.data)` 这样的 C 语言代码：它在 `itable` 中找到合适的函数指针，并将接口值的数据当作函数的第一个参数传入。你可以使用 `8g -S x.go` 来查看生成的代码（译者注：现在的命令已经变更为：`go tool compile -S x.go`）。请注意，在调用 `itable` 中的方法时传入的是接口值的 `s.data` 指针，而不是它实际指向的值。一般来说，在接口的调用端并不知道 `s.data` 的实际含义以及它里边到底存了什么。相反的，接口代码的实现使得 `itable` 中的函数指针仅期望一个 `32-bit` 形式的参数值。因此，示例中的函数指针实际上是 `(*Binary).String` 而不是 `Binary.String`。

目前的示例只考虑了接口只有一个方法的情况。拥有多个方法的接口，在其 `itable` 中会有同样数量的函数指针。

## itable 的计算

现在我们已经知道 `itable` 长什么样子，但是它们是从哪儿来的？Go 语言的动态类型转换说明了并不是编译器或者链接器预先计算了它们：因为有太多的对应关系（接口类型和具体类型），而且其中的大多数并不需要。实际的做法是，编译器为每一个具体类型生成了类型描述的数据结构，比如：`Binary` `int` 或 `func(map[int]string)`。除了其他元数据之外，类型描述的数据结构中包含了一个该类型实现了哪些方法的列表。同样的，编译器也为每一个接口类型，如 `Stringer`，生成了一个不同的类型描述的数据结构，其中也同样包含了这个列表。接口的运行时通过检查具体类型的 `itable` 中的每个方法来计算接口的 `itable`。运行时会缓存它找到的每一个方法，所以这个计算工作只需要进行一次。

在我们的例子中，`Binary` 的 `itable` 中有两个方法，而 `Stringer` 的 `itable` 中只有一个。假设我们将接口的方法数定义为 `ni`，具体类型的方法数定义为 `nt`，那么整个查询映射的过程的时间复杂度为 `O(ni x nt)`，但有优化的空间。对这两个 `itable` 排序然后并行地处理它们，那么[映射过程](https://github.com/golang/go/blob/weekly.2009-11-17/src/pkg/runtime/iface.c#L98)的时间复杂度会降为 `O(ni + nt)`。

## 内存优化
在上面描述的实现中，有两种互补的方法可以优化内存空间的使用。

第一种方法，如果接口类型是空的，也就是不包含任何方法，那么 `itable` 除了保存类型之外就没有其他的作用。在这种情况下，`itable` 会被丢弃，而接口值中原本指向 `itable` 的第一个 `32-bit` 字转为直接指向其对应的类型：

{% asset_img gointer3.png%}

接口类型是否拥有方法，这是一个静态属性。因为在源码中的定义是有区别的：`interface{}` 及 `interface{ methods...}`，由此编译器就能区分它们是哪一种表现形式。

第二种方法，如果一个接口关联的值可以由一个机器字表示，那么就不用引入复杂的间接寻址或堆的分配。如果我们定义一个与 `Binary` 类似，但是内部实现是 `uint32` 的类型，那么接口值的实际数据值将直接保存在接口值的第二个 `32-bit` 字中：

{% asset_img gointer4.png%}

实际数据值是直接保存在接口值中还是使用指针来指向，取决于其类型的大小。编译器针对传入的字，会安排合适的 `itable` 中的函数来进行处理。如果接收类型满足一个字，那么就会直接使用它；如果不满足，则对其解引用。如上面这些图所示，`Binary` 的方法在 `itable` 表现为 `(*Binary).String`，而在 `Binary32` 中为 `Binary32.String`，而不是 `(*Binary32).String`。

当然，接口为空且其值能用一个 `32-bit` 字表示时，会同时得到两种方法的优化：

{% asset_img gointer5.png%}


## 方法查询性能

Smalltalk 语言和其他一些动态语言，会在每次方法调用的时候进行方法查找。为了速度，许多语言的实现都是这样的：在每个调用方使用一个单条目的缓存，通常就在指令流中。在一个多线程的程序中，这些缓存就需要小心地进行管理，因为多线程程序可能会同时进行调用。即使忽略竞态条件，缓存也会使得内存成为争夺的焦点。因为 Go 语言具有与动态方法查找一起进行的静态类型的优势，即数据值已经保存在接口之中，就可以将方法查找移到调用方这边。比如下列代码所示：

```go
1   var any interface{}  // initialized elsewhere
2   s := any.(Stringer)  // dynamic conversion
3   for i := 0; i < 100; i++ {
4       fmt.Println(s.String())
5   }
```

在 Go 语言中，`itable` 在第 `2` 行赋值时进行了计算；在随后第 `4` 行调用 `s.String()` 就只需进行内存获取操作，然后执行单条指令 `call` 即可。（译者注：这就是上文提到的将方法查找移到调用方）

与此形成对比的是，在其他动态语言如 Smalltalk 中，将会在第 `4` 行的循环中做很多无用的方法查找操作。由于缓存的存在会使得开销降低一些，但依然无法像单条 `call` 指令那样高效。

当然，这只是一篇博客文章，我没有任何的数字来为此论据背书，但是减少内存争夺确实是高度并行程序中的一大胜利，因为能将方法查找从密集的循环中移出。另外，我这里讨论的是一般架构，而不是具体的某个实现：将来可能还会有一些恒定因子优化（译者注：可参考 [Optimizing Compiler#Factors affecting optimization](https://en.wikipedia.org/wiki/Optimizing_compiler#Factors_affecting_optimization)）。

## 更多的信息

接口运行时的实现位于，[iface.c](https://github.com/golang/go/blob/weekly.2009-11-17/src/pkg/runtime/iface.c)。还有很多关于接口的事情没有提到（我们甚至还没看到包含 `pointer receiver` 的示例（译者注：如 `func (b *Binary) somefunc()`））以及 `type` 描述符（除了运行时，还增强了反射能力），这些会在将来的文章中讨论。

## 代码

配套的代码(`x.go`)：

```go
1  package main
2
3  import (
4    "fmt"
5    "strconv"
6  )
7
8  type Stringer interface {
9    String() string
10  }
11
12 type Binary uint64
13
14 func (i Binary) String() string {
15   return strconv.Uitob64(i.Get(), 2)
16 }
17
18 func (i Binary) Get() uint64 {
19   return uint64(i)
20 }
21
22 func main() {
23   b := Binary(200)
24   s := Stringer(b)
25   fmt.Println(s.String())
26 }
```

节选部分 `8g -S x.go` 的输出（译者注：`8g` 编译器已不存在，现在使用命令实现同等效果：`go tool compile -S x.go`）：

```asm
0045 (x.go:25) LEAL    s+-24(SP),BX
0046 (x.go:25) MOVL    4(BX),BP
0047 (x.go:25) MOVL    BP,(SP)
0048 (x.go:25) MOVL    (BX),BX
0049 (x.go:25) MOVL    20(BX),BX
0050 (x.go:25) CALL    ,BX
```

`LEAL` 指令加载 `s` 的地址到寄存器 `BX` 中。（`n(SP)` 表示在内存中 `SP+n` 的字。`0(SP)` 可缩写为 `(SP)`。）接下来的两条 `MOVL` 指令获取接口的第二个 `32-bit` 字到寄存器 `BP` 中，然后将其保存为函数调用的第一个参数，即 `0(SP)`。最后两条 `MOVL` 指令获取 `itable` 地址，然后获取指向 `itable` 中对应函数的地址，为最终的调用做准备。

## 译者注
在 Go 1.14.4 中，`strconv.Uitob64` 函数已不存在，下面是替换后的代码：

```go
1  package main
2
3  import (
4	  "fmt"
5	  "strconv"
6  )
7
8  type Stringer interface {
9	 String() string
10 }
11
12 type Binary uint64
13
14 func (i Binary) String() string {
15	 return strconv.FormatUint(i.Get(), 2)
16 }
17
18 func (i Binary) Get() uint64 {
19	 return uint64(i)
20 }
21
22 func main() {
23	 b := Binary(200)
24	 s := Stringer(b)
25	 fmt.Println(s.String())
26 }
```

对应的 `go tool compile -s` 的汇编代码（已移除垃圾回收相关的指令）：
```asm
00046 (x.go:25)	LEAQ	go.itab."".Binary,"".Stringer(SB), AX
00053 (x.go:25)	TESTB	AL, (AX)
00055 (x.go:24)	MOVQ	8(SP), AX
00060 (x.go:25)	MOVQ	go.itab."".Binary,"".Stringer+24(SB), CX
00067 (x.go:25)	MOVQ	AX, (SP)
00071 (x.go:25)	CALL	CX
```
