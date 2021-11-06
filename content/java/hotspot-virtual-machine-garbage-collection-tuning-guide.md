---
title: HotSpot 虚拟机垃圾回收调优指南
date: 2021-06-21 23:48:28
categories: Java
draft: true
tags:
- java
- jvm
---

> - 原文地址：[Oracle Java Doc](https://docs.oracle.com/en/java/javase/11/gctuning/preface.html)
>
> - 对应版本：Java 11
>
> - 原文发表于：2018-09

## 垃圾回收调优简介
从桌面上的小程序到大型服务器上的 Web 应用，多种多样的程序都使用 Java 进行开发。为了支持这些部署形态和运行模式，Java HotSpot 虚拟机有针对性地为不同的场景提供了不同的垃圾回收器。Java 根据应用程序运行的计算类型来自动选择最恰当的垃圾回收器，然而这个选择有时并不是最佳的。有着严格性能目标或其他需求的用户、开发者及管理员需要显式地指定垃圾回收器，并通过调整配置参数以使得垃圾回收器达到期望的性能标准。本文档将提供信息来帮助完成这些操作。

<!-- more -->
