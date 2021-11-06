---
title: "JVM规范——指令集概述"
date: 2019-04-14T21:35:11+08:00
draft: false
categories:
  - JVM
tags:
  - Java
  - JVM
---

一个 JVM 指令由两部分组成，第一部分是一个字节（one-byte）的操作码，第二部分是 0 个或多个提供参数或数据的操作数，许多指令都只有第一部分。

<!--more-->

在忽略异常处理的情况下，JVM 解释器通常是在做如下循环：
```java
do {
    自动计算 PC 寄存器,并从 PC 寄存器获取操作码
    如果还有操作数,那么获取操作数
    执行操作
} while (还有更多可以做);
```
操作数的数量由操作码决定。如果一个操作数的大小大于一个字节，那么使用大端序保存。字节码指令流是单字节（single-byte）对齐的。有两个指令例外：`lookupswitch` 和 `tableswitch`，它们被强制为 4 字节（4-byte）对齐。

## 1. 类型与 JVM
JVM 指令集中的大部分指令编码与它们执行的操作数据类型有关，如： `iload` 指令读取局部变量的 `int` 值并压入操作数栈中。`fload` 指令对 `float` 类型做了同样的动作。两个指令实现了同样的功能，但是操作码却不同。

对于多数类型指定的指令，指令的类型已经在操作码的字面定义中体现：`i` 对应 `int` 操作，`l` 对应 `long`，`s` 对应 `short`，`b` 对应 `byte`，`c` 对应 `char`，`f` 对应 `float`，`d` 对应 `double`，`a` 对应引用类型。某些类型的指令明确说明类型不在命名中体现，比如 `arraylength` 指令始终对一个数组对象进行操作。其他一些指令，例如 `goto` ，一个无条件控制转移，不对类型指定的操作数进行操作。

限定操作数只为一个字节的大小，对 JVM 的指令集的设计也带来了压力。如果每个类型化指令都支持所有Java虚拟机的运行时数据类型，那么就会有比一个字节所能表示的更多的指令。相反，JVM 的指令集为某些操作提供了减少的类型支持。换句话说，指令集是故意不正交的。根据需要，可以使用单独的指令在不支持的和受支持的数据类型之间进行转换。

下面的表格总结了 JVM 指令集支持的类型，如果有些类型对应的操作码是空的，则说明没有支持该类型的指令。比如：`int` 类型有 `iload` 指令，而 `byte` 类型却没有对应的指令。请注意，下面表格中很多操作码都没有与 `byte` `char` `short` 相关的对应，甚至 `boolean` 类型都没有在表格里。编译器将 `byte` 和 `short` 类型通过符号扩展成为了 `int`，`boolean` 和 `char` 通过零扩展成为了 `int`。也就是说，`boolean` `byte` `char` `short` 的大多数操作都被当做 `int` 来对待。

>  符号扩展：低位数转高位数，在低位数的左边补上低位数的符号位，比如将一个字节扩展为两个字节：0b0000_1010 通过符号扩展成为 0b0000_0000_0000_1010，0b1000_0001 通过符号扩展成为 0b1111_1111_1000_0001。

> 零扩展：低位数转高位数，在低位数的左边全补 0，比如：0b0000_1010 通过零扩展成为 0b0000_0000_0000_1010，0b1000_0001 通过零扩展成为 0b0000_0000_1000_0001。

<table summary="Type support in the Java Virtual Machine instruction set" border="0">
                        <thead>
                           <tr>
                              <th>opcode</th>
                              <th><code class="literal">byte</code></th>
                              <th><code class="literal">short</code></th>
                              <th><code class="literal">int</code></th>
                              <th><code class="literal">long</code></th>
                              <th><code class="literal">float</code></th>
                              <th><code class="literal">double</code></th>
                              <th><code class="literal">char</code></th>
                              <th><code class="literal">reference</code></th>
                           </tr>
                        </thead>
                        <tbody>
                           <tr>
                              <td><span class="emphasis"><em>Tipush</em></span></td>
                              <td><span class="emphasis"><em>bipush</em></span></td>
                              <td><span class="emphasis"><em>sipush</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tconst</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>iconst</em></span></td>
                              <td><span class="emphasis"><em>lconst</em></span></td>
                              <td><span class="emphasis"><em>fconst</em></span></td>
                              <td><span class="emphasis"><em>dconst</em></span></td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>aconst</em></span></td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tload</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>iload</em></span></td>
                              <td><span class="emphasis"><em>lload</em></span></td>
                              <td><span class="emphasis"><em>fload</em></span></td>
                              <td><span class="emphasis"><em>dload</em></span></td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>aload</em></span></td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tstore</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>istore</em></span></td>
                              <td><span class="emphasis"><em>lstore</em></span></td>
                              <td><span class="emphasis"><em>fstore</em></span></td>
                              <td><span class="emphasis"><em>dstore</em></span></td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>astore</em></span></td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tinc</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>iinc</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Taload</em></span></td>
                              <td><span class="emphasis"><em>baload</em></span></td>
                              <td><span class="emphasis"><em>saload</em></span></td>
                              <td><span class="emphasis"><em>iaload</em></span></td>
                              <td><span class="emphasis"><em>laload</em></span></td>
                              <td><span class="emphasis"><em>faload</em></span></td>
                              <td><span class="emphasis"><em>daload</em></span></td>
                              <td><span class="emphasis"><em>caload</em></span></td>
                              <td><span class="emphasis"><em>aaload</em></span></td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tastore</em></span></td>
                              <td><span class="emphasis"><em>bastore</em></span></td>
                              <td><span class="emphasis"><em>sastore</em></span></td>
                              <td><span class="emphasis"><em>iastore</em></span></td>
                              <td><span class="emphasis"><em>lastore</em></span></td>
                              <td><span class="emphasis"><em>fastore</em></span></td>
                              <td><span class="emphasis"><em>dastore</em></span></td>
                              <td><span class="emphasis"><em>castore</em></span></td>
                              <td><span class="emphasis"><em>aastore</em></span></td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tadd</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>iadd</em></span></td>
                              <td><span class="emphasis"><em>ladd</em></span></td>
                              <td><span class="emphasis"><em>fadd</em></span></td>
                              <td><span class="emphasis"><em>dadd</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tsub</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>isub</em></span></td>
                              <td><span class="emphasis"><em>lsub</em></span></td>
                              <td><span class="emphasis"><em>fsub</em></span></td>
                              <td><span class="emphasis"><em>dsub</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tmul</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>imul</em></span></td>
                              <td><span class="emphasis"><em>lmul</em></span></td>
                              <td><span class="emphasis"><em>fmul</em></span></td>
                              <td><span class="emphasis"><em>dmul</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tdiv</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>idiv</em></span></td>
                              <td><span class="emphasis"><em>ldiv</em></span></td>
                              <td><span class="emphasis"><em>fdiv</em></span></td>
                              <td><span class="emphasis"><em>ddiv</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Trem</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>irem</em></span></td>
                              <td><span class="emphasis"><em>lrem</em></span></td>
                              <td><span class="emphasis"><em>frem</em></span></td>
                              <td><span class="emphasis"><em>drem</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tneg</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>ineg</em></span></td>
                              <td><span class="emphasis"><em>lneg</em></span></td>
                              <td><span class="emphasis"><em>fneg</em></span></td>
                              <td><span class="emphasis"><em>dneg</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tshl</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>ishl</em></span></td>
                              <td><span class="emphasis"><em>lshl</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tshr</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>ishr</em></span></td>
                              <td><span class="emphasis"><em>lshr</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tushr</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>iushr</em></span></td>
                              <td><span class="emphasis"><em>lushr</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tand</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>iand</em></span></td>
                              <td><span class="emphasis"><em>land</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tor</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>ior</em></span></td>
                              <td><span class="emphasis"><em>lor</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Txor</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>ixor</em></span></td>
                              <td><span class="emphasis"><em>lxor</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>i2T</em></span></td>
                              <td><span class="emphasis"><em>i2b</em></span></td>
                              <td><span class="emphasis"><em>i2s</em></span></td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>i2l</em></span></td>
                              <td><span class="emphasis"><em>i2f</em></span></td>
                              <td><span class="emphasis"><em>i2d</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>l2T</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>l2i</em></span></td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>l2f</em></span></td>
                              <td><span class="emphasis"><em>l2d</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>f2T</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>f2i</em></span></td>
                              <td><span class="emphasis"><em>f2l</em></span></td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>f2d</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>d2T</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>d2i</em></span></td>
                              <td><span class="emphasis"><em>d2l</em></span></td>
                              <td><span class="emphasis"><em>d2f</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tcmp</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>lcmp</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tcmpl</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>fcmpl</em></span></td>
                              <td><span class="emphasis"><em>dcmpl</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Tcmpg</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>fcmpg</em></span></td>
                              <td><span class="emphasis"><em>dcmpg</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>if_TcmpOP</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>if_icmpOP</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>if_acmpOP</em></span></td>
                           </tr>
                           <tr>
                              <td><span class="emphasis"><em>Treturn</em></span></td>
                              <td>&nbsp;</td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>ireturn</em></span></td>
                              <td><span class="emphasis"><em>lreturn</em></span></td>
                              <td><span class="emphasis"><em>freturn</em></span></td>
                              <td><span class="emphasis"><em>dreturn</em></span></td>
                              <td>&nbsp;</td>
                              <td><span class="emphasis"><em>areturn</em></span></td>
                           </tr>
                        </tbody>
                     </table>

某些 JVM 指令并不在意类型，比如 `pop` 和 `swap` 指令，它们直接操作操作数栈，而无论栈中的数据类型如何。

## 2. 加载和存储指令
加载和存储指令在栈帧里的局部变量与操作数栈之间进行传递：

* 加载一个局部变量到操作数栈：`iload` `iload_<n>` `lload` `lload_<n>` `fload` `float_<n>` `dload` `dload_<n>` `aload` `aload_<n>`
* 将操作数栈中的值保存到局部变量中：`istore` `istore_<n>` `lstore` `lstore_<n>` `fstore` `fstore_<n>` `dstore` `dstore_<n>` `astore` `astore_<n>`
* 将一个常量加载到操作数栈：`bipush` `sipush` `ldc` `ldc_w` `ldc2_w` `aconst_null` `iconst_m1` `iconst_<i>` `lconst_<i>` `fconst_<f>` `dconst_<d>`
* 使用更宽的索引访问更多的局部变量，或者访问更大的直接操作数：`wide`

>  `iload_0` 和 `iload` 的意思都是是将局部变量表中的下标 0 条目加载到操作数栈，`istore_2` 的意思是将操作数栈顶元素弹出并保存到局部变量表下标为 2 的条目。

## 3. 计算指令
计算指令弹出并计算操作数栈栈顶的两个值，然后将结果压回操作数栈，包括如下指令：

* 加：`iadd` `ladd` `fadd` `dadd`
* 减：`isub` `lsub` `fsub` `dsub`
* 乘：`imul` `lmul` `fmul` `dmul`
* 除：`idiv` `ldiv` `fdiv` `ddiv`
* 求余：`irem` `lrem` `frem` `drem`
* 取反：`ineg` `lneg` `fneg` `dneg`
* 位移：`ishl` `ishr` `iushr` `lshl` `lshr` `lushr`
* 按位或：`ior` `lor`
* 按位与：`iand` `land`
* 按位异或：`ixor` `lxor`
* 局部变量递增：`iinc`
* 比较：`dcmpg` `dcmpl` `fcmpg` `fcmpl` `lcmp`

## 4. 类型转换指令
类型转换指令允许 JVM 的两个数字类型进行互转，包括以下：

* 低位数转高位数：
    * `int` 转 `long` 或 `float` 或 `double`：`i2l` `i2f` `i2d`
    * `long` 转 `float` 或 `double`：`l2f` `l2d`
    * `float` 转 `double`：`f2d`
* 高位数转低位数：
    * `int` 转 `byte` `short` `char`：`i2b` `i2s` `i2c`
    * `long` 转 `int`：`l2i`
    * `float` 转 `int` `long`：`f2i` `f2l`
    * `double` 转 `int` `long` `float`：`d2i` `d2l` `d2f`

## 5. 对象的创建与操作
虽然类实例和数组都是对象，但是 JVM 创建和操作它们的时候使用了不同的指令集：

* 创建类实例：`new`
* 创建数组：`newarray` `anewarray` `multianewarray`
* 访问类属性（static field）：`getstatic` `putstatic`
* 访问成员属性：`getfield` `putfield`
* 加载数组中的元素到操作数栈：`baload` `caload` `saload` `iaload` `laload` `faload` `daload` `aaload`
* 将操作数栈元素保存到数组中某个元素：`bastore` `castore` `sastore` `iastore` `lastore` `fastore` `dastore` `aastore`
* 获取数组的长度：`arraylength`
* 检查实例类型：`instanceof` `checkcast`

## 6. 操作数栈管理指令
`pop` `pop2` `dup` `dup2` `dup_x1` `dup2_x1` `dup_x2` `dup2_x2` `swap`

## 7. 控制转移指令
控制转移指令将使 JVM 的执行转向对应的指令，而不是控制转移指令之后的指令：

* 条件转移：`ifeq` `ifne` `iflt` `ifle` `ifgt` `ifge` `ifnull` `ifnonnull` `if_icmpeq` `if_icmpne` `if_icmplt` `if_icmple` `if_icmpgt` `if_icmpge` `if_acmpeq` `if_acmpne`
* 符合条件分支：`tableswitch` `lookupswitch`
* 无条件转移：`goto` `goto_w` `jsr` `jsr_w` `ret`

JVM 在条件转移指令集中区分了 `int` 和引用类型。`boolean` `byte` `char` `short` 使用 `int` 指令集进行比较。`long` `float` `double` 的比较使用《3.计算指令》中描述的比较指令先得到一个 `int` 结果，再根据 `int` 结果的分支进行跳转。

## 8. 方法调用和返回指令
由下面 5 中指令来调用方法：

* `invokevirtual`：调用对象实例的一个指定方法，这是 Java 语言中常见的方法调用；
* `invokeinterface`：调用接口方法，搜索运行时实现了该接口方法的对象，并调用；
* `invokespecial`：调用特殊方法，比如实例化方法 `<init>` 和 `<clinit>`、私有方法或超类中的方法；
* `invokestatic`：调用类的静态方法；
* `invokedynamic`：调用 `Lambda` 动态绑定方法。
方法的返回指令分别为：`ireturn` `lreturn` `freturn` `dreturn` `areturn` `return`。

## 9.  抛出异常
`athrow`

## 10. 同步 指令
使用 `synchronized` 修饰的语句块：`monitorenter` `monitorexit`
