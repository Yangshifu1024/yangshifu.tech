---
title: "Java 内存模型基础"
date: 2019-04-03T23:50:07+08:00
draft: false
categories:
  - JVM
tags:
  - Java
  - JVM
  - JMM
---

在多核的计算机系统中，处理器通常拥有一个或多个处理器高速缓存（如 L1、L2 和 L3 等），用来提高数据的读取速度（因为 L1、L2 和 L3 等高速缓存从物理级别更接近处理器）和减少共享数据在总线上的传输（因为大部分的内存操作基本都能在 CPU 的本地高速缓存完成）。高速缓存可以极大地提高性能，但是也带来了新的挑战。例如：两个处理器同时访问一个内存地址，在何种条件下他们都能看到内存中同样的值？

<!--more-->

在处理器层面，内存模型定义了**其他处理器的写入对于当前处理器可见，及当前处理器的写入对其他处理器可见**的充要条件。一些处理器使用的是强内存模型，任何时间任意处理器所看到的任意内存地址的值都是一致的。还有一些处理器使用了较弱的内存模型，主要通过一些特殊的指令，也称为内存屏障，来实现将本地内存中的数据刷到主存或让这些数据无效（标记数据无效会使得处理器必须从主存读取该数据），使得其中一个处理器的写操作对于其他处理器都是可见的。这些内存屏障通常在执行锁定和解锁操作时执行，对于高级语言的程序员来说，它们是不可见的。

因为减少了对内存屏障的依赖，在强内存模型下更容易编写程序。但即使是在最强的内存模型下，内存屏障也是必须的；近年来，处理器的设计趋势更倾向于较弱的内存模型，因为更松散的内存一致性要求更有利于大规模的多处理器、高内存的系统扩展。

由于编译器对代码重排序，一个写操作能否对其他线程可见成为了问题。在不影响代码语义的情况下，编译器就会将代码重排序。由于缓存的影响，如果编译器重排序时将一个写操作滞后，那么其他线程将只能在这个写操作完成后才能看到其结果。

此外，对内存的写也可以排在前面，在这种情况下，其他线程将会在代码实际执行之前就看到其结果，这里的实际执行指的就是源码中的顺序，而实际执行的顺序是重排序后的顺序。所有的这些灵活性的设计，都是为了给编译器、运行时或者硬件在最佳的顺序下灵活地执行操作，在内存模型的限制之下，我们能实现更高的性能。

## Java 内存模型（Java Memory Model, JMM)
JMM 描述了什么样的行为在多线程代码中是合法的，以及线程如何通过内存进行交互；描述了程序中变量之间的关系；也描述了在底层中数据从内存或寄存器的存取细节。在多种硬件及多种形式的编译器优化下依然能够保持程序的正确性。

Java 包括几种语言结构：`volatile`，`final` 和 `synchronized`，用以帮助程序开发者正确地为编译器描述程序的并发依赖。JMM 定义了 ``volatile`` 和 ``synchronized`` 的行为，更重要的是，其保证了一个正确的同步程序在多处理器架构上也能正确运行。

## 原则

### 原子性
在一个线程中的一系列操作，其中间过程不会被外部知晓，外部只能看到最终结果，而无法对其中间过程进行干扰。

### 可见性
在一个线程中对共享变量对修改，在另外一个线程中能够立即看到。

### 有序性
编译优化、处理器执行优化均不能改变语句的语义。

## 同步

线程之间的通信机制有两种：共享内存和消息传递。

Java 采用的是共享内存模型。

## 重排序

### 1. 编译器优化的重排序
编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。

### 2. 指令级并行重排序
现代处理器采用了指令级并行技术(Instruction-Level Parallelism, ILP)来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。

### 3. 内存系统的重排序
由于处理器使用缓存和读写缓冲区，这使得加载和存储操作看上去像是在乱序执行。

为了保证内存可见性，Java 编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。

## 内存屏障
JMM 把内存屏障指令分为 4 类：

### 1. LoadLoad Barriers
 **Load1;LoadLoad;Load2** 确保 Load1 数据的装载先于 Load2 及所有后续装载指令的装载；
### 2. StoreStore Barriers
**Store1;StoreStore;Store2** 确保 Store1 数据对其他处理器可见（刷新到内存）先于 Store2 及所有后续存储指令对存储；
### 3. LoadStore Barriers
**Load1;LoadStore;Store2** 确保 Load1 数据装载先于 Store2 及所有后续的存储指令刷新到内存；
### 4. StoreLoad Barriers
**Store1;StoreLoad;Load2** 确保 Store1 数据对其他处理器变得可见（刷新到内存）先于 Load2 及所有后续装载指令的装载。该屏障会使在其之前的所有内存访问指令（存储和装载指令）完成之后，才执行该屏障之后的内存访问指令。

其中，**StoreLoad Barriers** 是一个全能型屏障，它同时具有其他 3 个屏障的效果。而执行该屏障开销会很昂贵，因为当前处理器通常要把写缓冲区中的数据全部刷新到内存中（Buffer Fully Flush）。

## Happens-Before

Happens-Before 指的是：**前一个操作的结果对后续操作可见**。

与程序员密切相关的 7 条规则如下：

1. 程序顺序规则：一个线程中的每个操作，均 Happens-Before 于该线程中的任意后续操作；
2. 监视器锁规则：Monitor Lock，通常是 `synchronized` 关键字，对于一个锁的解锁，均 Happens-Before 于随后对这个锁对加锁；
3. Volatile 变量规则：对于一个 `volatile` 变量对写操作，Happens-Before 于任意后续对这个 `volatile` 变量的读；
4. 线程启动规则：Thread 对象的 `Thread#start()` 方法 Happens-Before 于此线程的每一个动作；
5. 线程终止规则：线程中的所有操作均 Happens-Before 于该线程都终止；
6. 线程中断规则：对线程的 `Thread#interrupt()` 方法的调用 Happens-Before 于该线程中的代码检测到中断事件的发生；
7. 对象终结规则：一个对象的初始化完成（构造函数执行结束）Happens-Before 于它的 `Object#finalize()` 方法的开始。

Happens-Before 的特性传递性：

如果 A Happens-Before B，且 B Happens-Before C，那么 A Happens-Before C。

## 数据竞争

JMM 规范对数据竞争对定义如下：

1. 在一个线程中写一个变量；
2. 在另一个线程中读同一个变量；
3. 读和写都没有通过同步来排序。

一个有数据竞争的程序就是**没有正确同步的有问题的**程序。

## 顺序一致性

顺序一致性内存模型有两大特性：

1. 一个线程中都所有操作必须按照程序都顺序来执行；
2. 所有线程都只能看到一个单一的操作执行顺序，且线程中的每个操作都必须原子执行且立刻对所有线程可见。

假设有两个线程 A 和 B 并发执行。A 线程中有三个操作，它们的顺序为： A1-A2-A3；B 线程中也有三个操作，它们的顺序为：B1-B2-B3。

- 假设两个线程使用锁来进行正确的同步，如 synchronized 关键字或显式声明的锁等，那么两个线程的执行顺序为：

```
A1->A2->A3->B1->B2->B3
```

- 假设两个线程没有进行同步，那么它们的执行顺序可能为：

```
B1->A1->B2->A2->A3->B3
```

虽然它们的执行顺序是无序的，但两个线程都只能看到一个一致的整体执行顺序，即在两个线程中，都是如上所示的顺序。

但以上这个情况在 JMM 没有得到保证，在 JMM 中，两个线程整体的执行顺序是无序的，而且两个线程看到的操作执行顺序也可能不一致。比如，在 A 线程中将写过的数据缓存在本地内存中，在没有刷新到主存之前，从 B 线程中观察，会认为这个写操作根本没有执行。
