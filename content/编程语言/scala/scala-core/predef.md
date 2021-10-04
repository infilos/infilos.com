---
type: docs
title: "Predef"
linkTitle: "Predef"
weight: 29
---

**Predef 提供了一些定义，可以在所有 Scala 的编译单元中可见且不需要明确的限制。在所有的 Scala 代码中自动引入。**

## 最常用的类型

提供了一些最常用类型的类型别名(alias)。比如一些不可变集合及其构造器。

## 控制台 I/O

提供了一些用于控制台 I/O 的函数，比如：`print`、`println`等。这些函数都是`scala.Console`中提供的函数的别名。

## 断言

一组`assert`函数用于注释和动态检查代码中的常量。

> 在命令行中添加参数`-Xdisable-assertions`可以完成编译器的`assert`调用。

## 隐式转换

这里和其父类型`scala.LowPriorityImplicits`提供了一组最常用的隐式转换。为一些类型提供了一些扩展功能。