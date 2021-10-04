---
type: docs
title: "占位符"
linkTitle: "占位符"
weight: 38
---

Scala 的匿名函数语法`arg => expr`，提供来非常简洁的方式来构造函数字面值，甚至函数中包含**多个语句**。

同时，匿名函数中可以使用占位符语法：

```scala
List(1, 2).map { i => i + 1 }
// equivalent
List(1, 2).map { _ + 1 }
```

当时，如果在上面的例子中添加一个 debug 信息，比如：

```scala
List(1, 2).map { i => println("Hi"); i + 1 }
Hi 
Hi 
List[Int] = List(2, 3) 

List(1, 2).map { println("Hi"); _ + 1 }
Hi 
List[Int] = List(2, 3)
```

可以发现，结果并不符合预期。

因为函数经常被当做参数传递，经常可以看到他们被`{...}`包围。通常会认为大括号表示一个匿名函数，但是它只是一个**块表达式**：一条或多条语句，最后一条决定了这个块的结果。

上面的例子中，两个块被解析的方式不同，决定了他们的不同行为。

第一条语句中，`{ i => println("Hi"); i + 1 }`被认为是一个`arg => expr`方式的**一个**函数字面值语句，而`expr`在这里就是`println("Hi"); i + 1`。因此，`println`语句也是函数体的一部分，每当函数被调用时，他都会被执行。

```scala
scala> val printAndAddOne = (i: Int) => { println("Hi"); i + 1 } 
printAndAddOne: Int => Int = <function1>

scala> List(1, 2).map(printAndAddOne) 
Hi 
Hi 
res29: List[Int] = List(2, 3)
```

第二条语句中，代码块被识别为**两个**表达式，`println("Hi")`和`_ + 1`，`println`语句并不是函数体的一部分，它会在整个语句块`{ println("Hi"); _ + 1 }`当做参数传递给 map 方法时执行，而不是 map 方法执行的时候。而整个块的计算结果，即最后一行语句的值，`_ + 1`，作为一个`Int => Int`的匿名函数传递给 map 方法。

```scala
scala> val printAndReturnAFunc = { println("Hi"); (_: Int) + 1 } 
Hi 												// println 语句已经被调用
printAndReturnAFunc: Int => Int = <function1>	// 整个块已经计算完成，得到匿名函数

scala> List(1, 2).map(printAndReturnAFunc) 
res30: List[Int] = List(2, 3)
```

## 总结

这个地方的关键是：**使用占位符定义的匿名函数的作用域仅延伸到包含占位符(_)的表达式**；**普通语法的匿名函数，其函数体包含从标示符(=>)开始直到语句块结束**。

普通语法的匿名函数：

```scala
scala> val regularFunc = { a:Any => println("foo"); println(a); "baz"}
regularFunc: Any => String = <function1>

scala> regularFunc("hello")
foo
hello
res0: String = baz
```

占位符语法的匿名函数，下面这两个函数是等效的：

```scala
scala> val anonymousFunc = { println("foo"); println(_: Any); "baz" }
foo 
anonymousFunc: String = baz

scala> val confinedFunc = { println("foo"); { a: Any => println(a) }; "baz" } 
foo 
confinedFunc: String = baz
```

