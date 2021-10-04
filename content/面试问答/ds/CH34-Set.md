---
type: docs
title: "CH34-Set"
linkTitle: "CH34-Set"
weight: 34
---

因为 Set 的结构及实现都和 Map 保持高度一致，这里将不再对其进行分析了，感兴趣的朋友可以自行查看源码。但我们还是需要知道什么是 Set，Set 是一个包含不可重元素的集合，也就是所有的元素都是唯一的。还是看下文档说明吧：

> A collection that contains no duplicate elements.  More formally, sets contain no pair of elements e1 and e2 such that e1.equals(e2), and at most one null element.  As implied by its name, this interface models the mathematical set abstraction.

此外 Set 系列也有 SortedSet、NavigableSet 这种基于排序的接口，它们的作用在分析 Map 时都已经详细介绍过了。

