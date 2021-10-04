---
type: docs
title: "String"
linkTitle: "String"
weight: 17
---

## 介绍

Scala 中的 String 其实是 Java 中的 String，因此可以在 Scala 中使用所有 Java 中的相关 API。同时，StringOps 中定义了很多 String 相关的隐式转换，因此可以对 String 使用很多方便的操作，或者将 String 看做是一个有**字符(character)**组成的序列来操作，使其拥有序列的所有方法。

```scala
// Predef.scala
type String        = java.lang.String

// Usage
"hello".foreach(println)
for(c <- "hello") println(c)
s.getBytes.foreach(println) // 104,101,108...
"hello world".filter(_ != 'l')
```

### StringOps

在 Scala 中，会自动引入 Predef 对象，Predef 中定义了 String 到 StringOps 的隐式转换，根据需要，String 会被 隐式转换为 StringOps 以拥有所有**有序序列**的方法。

```scala
final class StringOps extends AnyVal with StringLike[String] {
  override protected[this] def thisCollection: WrappedString = new WrappedString(repr)
  override protected[this] def toCollection(repr: String): WrappedString = new WrappedString(repr)
  ...
  def seq = new WrappedString(repr)
}
new StringOps(repr: String)
```

### WrappedString

Predef 中还定义了 String 和 WrappedString：

```scala
implicit def wrapString(s: String): WrappedString = if (s ne null) new WrappedString(s) else null
implicit def unwrapString(ws: WrappedString): String = if (ws ne null) ws.self else null
```

WrappedString 的实现是对 String 的包装容器，将 String 作为一个参数并提供所有**有序序列**的操作。

```scala
class WrappedString extends AbstractSeq[Char] with IndexedSeq[Char] with StringLike[WrappedString]
new WrappedString(self: String)
```

### StringLike

WrappedString 与 StringOps 的不同是，当执行一些类似 filter、map 的转换操作时，该类生成的是一个 WrappedString 对象而不是 String，在 StringOps 中，间接使用了 WrappedString 的封装。二者都混入了 StringLike，区别在于前者的混入方式是 `StringLike[WrappedString]`，后者是`StringLike[String]`。

```scala
trait StringLike[+Repr] extends IndexedSeqOptimized[Char, Repr] with Ordered[String]
```

在 StringLike 特质中，实现了 String 对象的相关集合操作。

## 检查相等性

两个 String 的相等，实际上是检查两个字符集合的相等。

```scala
"hello" == "world"
"hello" == "hello"
"hello" == null		// ok
null == "hello"		// ok
```

当对一个值为 null 的 String 进行相等性检查时并不会出现空指针异常，但是当使用一个 null 调用方法时则会出现空指针异常：

```scala
val test = null
test.toUpperCace == "HELLO"	// java.lang.NullPointerException
```

忽略大小写的相等性检查：

```scala
a.equalsIgnoreCase(b)
```

> Scala 中检查对象相等使用的是`==`而不是 Java 中的`equal`。
>
> `x == y` 实际上是：`if (x eq null) x eq null else x.equals(y)`
>
> 因此，使用`==`进行相等判断时不需要检查是否为`null`。

## 创建多行 String

```scala
val foo = """This is 
a multiline 
String"""
  
val speech = """Four score and 
|seven years ago""".stripMargin
```

## 切分 String

```scala
"hello world".split(" ")
// Array(hello, world)
```

## 将变量带入 String

```scala
val name = ???
val age = ???
val weight = ???
println(s"$name is $age years old, and weighs $weight pounds.")
```

或者在 String 中使用表达式：

```scala
println(s"Age next year: ${age + 1}")
```

## 逐个处理 String 中的每个字符

```scala
val upper = "hello, world".map(c => c.toUpper)
val upper = "hello, world".map(_.toUpper)
```

同时可以使用集合的方法与字符串方法相结合：

```
val upper = "hello, world".filter(_ != 'l').map(_.toUpper)
```

## 模式查找

如果需要使用正则表达式来对 String 中需要的部分进行匹配，首先使用`.r`方法创建一个`Regex`对象，然后使用`findFirstIn`或`findAllIn`查找第一个或所有匹配的结果。

```scala
scala> val numPattern = "[0-9]+".r 
numPattern: scala.util.matching.Regex = [0-9]+

scala> val address = "123 Main Street Suite 101"

scala> val match1 = numPattern.findFirstIn(address) 
match1: Option[String] = Some(123)

scala> val matches = numPattern.findAllIn(address) 
matches: scala.util.matching.Regex.MatchIterator = non-empty iterator

scala> matches.foreach(println)
123
101
```

处理匹配的结果：

```scala
match1 match{
  case Some(result) => ???
  case None =>  ???
}

val match1 = numPattern.findFirstIn(address).getOrElse("no match")
```

或者另一种方式创建`Regex`对象：

```scala
import scala.util.matching.Regex
val numPattern = new Regex("[0-9]+")
```

> 对于正则表达式的使用，可以应用一个名为“JavaVerbalExpressions”的扩展库，以更 DSL 的方式构建`Regex`对象。
>
> VerbalExpression.regex()
>
> ```scala
> VerbalExpression.regex().startOfLine().then("http").maybe("s")
>                         .then("://")
>                         .maybe("www.").anythingBut(" ")
>                         .endOfLine()
>                         .build();
> ```

## 模式替换

由于 String 是不可变的，不可以在原有的 String 上进行修改，可以创建一个新的 String 包含替换后的结果。

```scala
val address = "123 Main Street".replaceAll("[0-9]", "x")

val regex = "[0-9]".r
val newAddress = regex.replaceAllIn("123 Main Street", "x")

val result = "123".replaceFirst("[0-9]", "x")
```

## 使用正则解析多个部分

如果需要将 String 的多个匹配部分解析到不同的变量，可以使用**正则表达式组**：

```scala
val pattern = "([0-9]+) ([A-Za-z]+)".r
val pattern(count, fruit) = "100 Bananas"

count: String = 100
fruit: String = Bananas
```

或者同事创建多种模式以匹配不同的预期结果：

```scala
"movies near 80301" 
"movies 80301" 
"80301 movies" 
"movie: 80301" 
"movies: 80301" 
"movies near boulder, co" 
"movies near boulder, colorado"

// match "movies 80301" 
val MoviesZipRE = "movies (\\d{5})".r

// match "movies near boulder, co" 
val MoviesNearCityStateRE = "movies near ([a-z]+), ([a-z]{2})".r

textUserTyped match { 
  case MoviesZipRE(zip) => getSearchResults(zip) 
  case MoviesNearCityStateRE(city, state) => getSearchResults(city, state) 
  case _ => println("did not match a regex") 
}
```

## 访问字符串中的字符

可以以位置索引来访问 String 中的字符：

```scala
"hello".charAt(0)
"hello"(0)
"hello".apply(1)
```

## 为 String 类添加额外的方法

如果通过给现有的 String 添加额外的方法，使 String 拥有需要的方法，而不是将 String 作为一个参数传入一个需要的方法：

```scala
"HAL".increment
// 而不是
StringUtilities.increment("HAL")
```

可以创建一个隐式类，然后在隐式类中添加需要的方法：

```scala
implicit class StringImprovements(s: String) {
  def increment = s.map(c => (c + 1).toChar)
}

val result = "HAL".increment
```

但是在真实的应用中，隐式类必须在一个 class、object、package 中定义。

### 在 object 中定义 隐式类

```scala
object StringUtils{
  implicit class StringImprovements(val s:String){
    def increment = s.map(c => (c + 1).toChar)
  }
}
```

### 在 package 中定义 隐式类

```scala
package object utils{
  implicit class StringImporvemrnts(val s:String){
    def increment = s.map(c => (c +1).toChar)
  }
}
```

### 使用隐式转换的方式

首先定义一个类，带有一个需要的方法，然后创建一个隐式转换，将 String 转换为这个带有目的方法的对象，就可以在 String 上调用该方法了：

```scala
class StringImprovement(val s: String){
  def increment = s.map(c => (c +1).toChar)
}

implicit def stringToStringImpr(s:String) = new StringImprovement(s)
```

