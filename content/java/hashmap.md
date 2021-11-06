---
title: "HashMap"
date: 2019-04-22T22:00:52+08:00
draft: true
categories:
  - Java
tags:
  - collections
---

## 基本描述

`HashMap` 是一个基于哈希表(Hash table)的 `Map` 接口的实现，它实现了 `Map` 接口的所有操作，并允许空值和空键。

它除了非线程安全和允许空值空键外，大致和 `Hashtable` 是一致的；并且它也不能保证其中元素的顺序，实际上，随着时间的推移，其中元素的顺序也会发生改变。


<!-- more -->

在元素没有冲突、所有元素都能够均匀地分散到其中的各个 `bucket` 的情况下，`HashMap` 基本的 `get` 和 `put` 操作的时间复杂度是常量的，即 O(n)。

对 `HashMap` 实例迭代的时间与它的容量 (`bucket` 的数量) 加上大小 (`bucket` 的大小) 成正比。因此，如果需要较好的迭代性能，就不要将初始的容量 (`capaticy`) 设置得过高，或者负载系数 (`load factor`) 设置得过低。

`HashMap` 实例的两个参数将直接影响它的性能，初始化容量 (`capacity`) 和负载系数 (`load factor`)。容量指的是 `HashMap` 内部桶 (`bucket`) 的个数，而负载系数 (`load factor) 则决定了 `HashMap` 进行扩容的临界值。如果 `HashMap` 中元素的实际个数 `size` 超过了容量和负载系数的乘积，那么 `HashMap` 就会重新进行散列 (`rehashed`)，内部的数据结构将被重建为接近原来的两倍。

按照惯例，负载系数的默认值 `0.75` 在时间和空间的开销上取得了最佳权衡。更高的值减少了空间的浪费，却提高了查表的开销，影响了大部分的 `HashMap` 操作，包括 `get` 和 `put`。在设置 `HashMap` 的初始容量时，应考虑预期的元素个数和负载系数，为其设置合适的初始值，减少在使用过程中重新进行散列 (`rehashed`) 的操作。如果元素的最大个数始终小于初始容量与负载系数的乘积，则不会重新进行散列。

如果 `HashMap` 中要存储很多元素，那么在创建 `HashMap` 实例时就应该使用足够大的初始化容量，这样的情况下存储将更加有效，因为不需要进行重新散列(`rehash`)来扩展。当 `HashMap` 中的键 `Key` 的 `hashCode()` 方法返回值相同时，将严重降低 `HashMap` 的性能，所以我们可以重写键类的 `hashCode()` 方法来进行优化，或者实现 `Comparable` 接口的 `compareTo()` 方法。

`HashMap` 不是线程安全的，如果多个线程同时进行读写，那么需要在外部保证其线程安全。这里的『写』包括添加和删除元素，不包括替换已有的元素。通常通过封装对 `HashMap` 的操作来实现同步（如：在对 `HashMap` 进行操作的方法加上 `synchronized` 或者使用别的同步手段）。

如果没有进行封装的对象，也可以在创建 `HashMap` 时使用 `Collections#synchronizedMap()` 方法来创建一个线程安全的实例：

```java
Map m = Collections.synchronizedMap(new HashMap(...));
```

`HashMap` 的迭代器在创建之后，只能通过 `iterator` 自己的 `remove` 方法对元素进行删除，除此以外的元素变动，都将导致使用 `iterator` 时抛出 `ConcurrentModificationException` 异常。

## 实现

`HashMap` 在内部是由桶 (`bin` 或 `bucket`) 组成的一个哈希表，当桶变大时，将转换为 `TreeNodes`，每个 `TreeNodes` 的结构与 `java.util.TreeMap` 相似。

每个桶里的 `TreeNodes` 元素的顺序也是通过 `hashCode` 来保证，如果某两个元素实现了 `Comparable` 接口，则使用它们的 `compareTo` 方法来排序。
