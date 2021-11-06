---
title: "JVM规范 —— 代码编译"
date: 2019-04-24T21:05:12+08:00
draft: true
categories:
  - Java
tags:
  - bytecode
  - compilation
---

“编译器”一词，有时也指将 JVM 指令集翻译为特定 CPU 架构指令集的工具，比如 JIT（just-in-time）即时编译器。但本文档讨论的范围仅包括将 Java 源代码编译为 JVM 指令。

<!--more-->

### 1.  格式示例
`javap` 工具用来反编译 Java 字节码，反编译后的指令示例如下：
```
<index> <opcode> [ <operand1> [<operand2>...] ] [<comment>]
```
`<index>` 是当前方法内指令数组所对应的下标，可以理解为从方法开始处以字节为单位的偏移。`<opcode>` 是指令操作码的助记符。0 个或多个 <operandN> 是指令的操作数。`<comment>` 则是可选的注释语法。比如以下示例：
```
8    bipush    100    // Push int constant 100
```

通常注释是由 `javap` 命令产生的。每个指令的前缀 `<index>` 可用作控制传输指令的目标，比如 `goto 8` 指令将控制权转移到所以为 8 的指令处。JVM 在实际进行控制传输时使用的操作数其实是相对于该指令的偏移地址，在这里 `javap` 将其显示地直观。

通常使用 `#` 来指定运行时常量池的索引，比如：
```
10    ldc    #1    // Push float constant 100.0
```
或者
```
9    invokevirtual    #4    // Method Example.addTwo(II)I
```

### 2. 常量、局部变量表及控制流程的使用

下面的 `spin` 方法简单地空循环100次：
```java
void spin() {
    int i;
    for (i = 0; i < 100; i++) {
        ;    // Loop body is empty
    }
}
```
编译器将代码编译为：
```
0   iconst_0       // Push int constant 0
1   istore_1       // Store into local variable 1 (i=0)
2   goto 8         // First time through don't increment
5   iinc 1 1       // Increment local variable 1 by 1 (i++)
8   iload_1        // Push local variable 1 (i)
9   bipush 100     // Push int constant 100
11  if_icmplt 5    // Compare and loop if less than (i < 100)
14  return         // Return void when done
```

JVM 是以栈（stack-oriented）为主导的，许多指令将所需的操作数从操作数栈取出，执行指令，然后将结果压回操作数栈中。每个方法执行时都创建一个栈帧，并在其中创建操作数栈和局部变量，以供方法使用。在程序执行的过程中，每个线程需要管理许多的栈帧和许多的操作数栈来与嵌套的方法调用对应。每个栈帧中同一时刻只会有一个操作数栈处于激活状态。

在 `spin` 方法中，有两个常量：`0` 和 `100`，在加载到操作数栈时使用了不同的指令，`0` 使用的是 `iconst_0` 指令，而 `100` 使用的是 `bipush` 指令，此指令将使得 `100` 成为立即操作数。

> 对于 `int` 值的不同，使用的指令也不同：取值 -1~5 采用 iconst 指令，取值 -128~127 采用 bipush 指令，取值 -32768~32767 采用 sipush 指令，取值 -2147483648~2147483647 采用 ldc 指令。

`int i` 保存在局部变量表中的第一个，也就是索引 1 的位置。因为 JVM 指令在执行时总是使用操作数栈中的值，而不是直接使用局部变量，所以值在局部变量表和操作数栈中的相互传递在编译后的字节码中很常见。在 `spin` 方法中，变量 `i` 的相互传递是通过 `istore_1` 和 `iload_1` 来完成的，它们都是对第 `1` 个局部变量进行操作：`istore_1` 的作用是将一个 `int` 值从操作数栈中弹出并保存在局部变量表索引 1 的位置，而 `iload_1` 则是将局部变量表索引 1 位置的值压入操作数栈中。
