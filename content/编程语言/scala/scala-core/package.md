---
type: docs
title: "Package"
linkTitle: "Package"
weight: 27
---

## 介绍

Scala 中会自动导入两个包：

- java.lang._
- scala._

这其中，包括 scala.Predef，预定义了一些常用的功能。

## 以大括号的方式定义包

```scala
package com.acme.store {
  class Foo { override def toString = "I am com.acme.store.Foo" } 
}
// 等同于
package com.acme.store 
class Foo { override def toString = "I am com.acme.store.Foo" }
```

使用这种大括号的方式可以在一个文件内定义多个包，或者嵌套的包。

## 引入一个或多个成员

```scala
import java.io.File
import java.io.{File, IOException, FileNotFoundException}
import java.io._
```

- 可以在任意位置引入成员，类中、对象中、方法或代码块中
- 可以引入任意成员，类、包、对象
- 可以隐藏或重命名引入的成员

最佳实践时：除非需要引入的对象超过3个，则一般不适用通配符引入，避免不必要的冲突。

## 重命名引入的成员

有时候引入的成员会和当前作用域中的成员名冲突，或者需要一个更有意义的名字，这时候可以将引入的成员重命名：

```scala
import java.util.{ArrayList => JavaList}
import java.util.{Date => JDate, HashMap => JHashMap}
```

但是重命名之后，就不能再使用原有的成员名了。

## 引入时隐藏部分成员

```scala
import java.util.{Random => _, _}
```

这个语法会引入除 Random 之外的所有包，仅仅是把 Random 隐藏了。

或者同时隐藏多个成员，只引入剩余的其他成员：

```scala
import java.util.{List => _, Map => _, Set => _, _}
```

## 使用静态引入

如果想要以 Java 静态引入的方式引入一个成员，以便能够直接引用成员的名字：

```scala
import java.lang.Math._
```

然后就可以使用 Math 中的所有成员，`sin(0)、cos(PI)`，而不再需要以`Match.sin(0)`的方式使用。

## 在任何地方引入

唯一需要注意的是，引入语句的位置必须处于使用的位置之前，否则会找不到使用的对象。

