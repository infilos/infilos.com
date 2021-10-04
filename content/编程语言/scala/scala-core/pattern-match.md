---
type: docs
title: "Pattern Match"
linkTitle: "Pattern Match"
weight: 19
---

## 类比 switch

模式匹配类似于 Java 中的`switch`语句，但与它不同之处在于：

- 它是一个表达式，有返回值
- 不会落空
- 没有任何匹配到的项则会抛出`MatchError`

## 基本语法

` case pattern => result`：

- `case`后跟一个`pattern`
- `pattern`必须是合法的模式类型之一
- 如果与模式匹配上了，结果会被计算并返回

## 通配符模式

```scala
def matchAll(any: Any): String = any match {
  case _ => "It’s a match!"
}
```

通配符`_`会对匹配所有对象，会当做一个默认的模式来使用，以避免`MatchError`。

## 常量模式

```scala
def isIt8(any: Any): String = any match {
  case "8:00" => "Yes"
  case 8 => "Yes"
  case _ => "No"		// 对其他任何情况进行匹配，设置一个默认的 No
}
```

## 变量模式

```scala
def matchX(any: Any): String = any match {
  case x => s"He said $x!"
}
```

这里的`X`作为一个标示符，会对任何对象进行匹配。

## 变量 & 常量

```scala
import math.Pi

val pi = Pi

def m1(x: Double) = x match {
  case Pi => "Pi!"				// 大写的常量模式，会引用常量 Pi 的值进行匹配
  case _ => "not Pi!"
}

def m2(x: Double) = x match {
  case pi => "Pi!"				// 小写的变量模式，所有的对象都会进行匹配
  case _ => "not Pi!"
}

println(m1(Pi))		// Pi!
println(m1(3.14))	// not Pi!

println(m2(Pi))		// Pi!
println(m2(3.14))	// Pi!
```

**这里需要注意的地方：**

- 如果标示符为**大写**，编译器会把它当做一个**常量**
- 如果标示符为**小写**，编译器会把它当做一个**变量**，仅限匹配表达式内部的作用域

如果需要在匹配表达式中引用一个变量的值，可以通过一下方式：

- 使用名称限制，比如`this.pi`
- 使用引号包围变量，比如 "\`pi\`"

```scala
import math.Pi

val pi = Pi

def m3(x: Double) = x match {
  case this.pi => "Pi!"
  case _ => "not Pi!"
}

def m4(x: Double) = x match {
  case `pi` => "Pi!"
  case _ => "not Pi!"
}
```

## 构造器模式 - case 类

```scala
case class Time(hours: Int = 0, minutes: Int = 0)
val (noon, morn, eve) = (Time(12), Time(9), Time(20))

def mt(t: Time) = t match {
  case Time(12,_) => "twelve something"
  case _ => "not twelve"
}
```

### 嵌套构造器

```scala
case class House(street: String, number: Int)
case class Address(city: String, house: House)
case class Person(name: String, age: Int, address: Address)

val peter = Person("Peter", 33, Address("Hamburg", House("Reeperbahn", 45)))
val paul = Person("Paul", 29, Address("Berlin", House("Oranienstrasse", 64)))

def m45(p: Person) = p match {
  case Person(_, _, Address(_, House(_, 45))) => "Must be Peter!"
  case Person(_, _, Address(_, House(_, _))) => "Someone else"
}
```

## 序列模式

```scala
val l1 = List(1,2,3,4)
val l2 = List(5)
val l3 = List(5,8,6,4,9,12)

def ml(l: List[Int]) = l match {
  case List(1,_,_,_) => "starts with 1 and has 4 elements"
  case List(5, _*) => "starts with 5"
}
```

### 另一种用法

```scala
import annotation._

@tailrec
def contains5(l: List[Int]): String = l match {
  case Nil => "No"
  case 5 +: _ => "Yes"
  case _ +: tail => contains5(tail)
}
```

这里的符号`+:`实际上是一个序列的解析器，其源码中的定义为：

```scala
/** An extractor used to head/tail deconstruct sequences. */
object +: {
  def unapply[A](t: Seq[A]): Option[(A, Seq[A])] =
    if(t.isEmpty) None
    else Some(t.head -> t.tail)
}
```

因此上面的匹配语句实际上等同于：

```scala
@tailrec
def contains5(l: List[Int]): String = l match {
  case Nil => "No"
  case +:(5, _) => "Yes"
  case +:(_, tail) => contains5(tail)
}
```

## 解析器

- 一个解析器是一个拥有`unapply`方法的 Scala 对象
- 可以吧`unapply`理解为`apply`的反向操作
- `unapply`会将需要匹配的值当做一个参数(如果这个值与`unapply`的参数类型一致)
- 返回结果：
  - 没有变量时返回：Boolean
  - 一个变量时返回：`Option[A]`
  - 多个变量时返回：`Option[TupleN[...]]`
- the returned is matched with your pattern

### 编写一个解析器

```scala
case class Time(hours: Int = 0, minutes: Int = 0)
val (noon, morn, eve) = (Time(12), Time(9), Time(20))

object AM {
  def unapply(t: Time): Boolean = t.hours < 12
}

def greet(t:Any) = t match {
  case AM() => "Good Morning!"	// 这里调用 AM 中的 unapply 方法，t 作为其参数传入
  case _ => "Good Afternoon!"
}
```

### 变量绑定

```scala
object AM {
  def unapply(t: Time): Option[(Int,Int)] = 
    if (t.hours < 12) Some(t.hours -> t.minutes) else None
}

def greet(t:Time) = t match {
  case AM(h,m) => f"Good Morning, it's $h%02d:$m%02d!"	// 将 t 的字段绑定到变量 h、m
  case _ => "Good Afternoon!"
}
```

### 未知数量的变量绑定

```scala
val s1 = "lightbend.com"
val s2 = "www.scala-lang.org"

object Domain {
	def unapplySeq(s: String) = Some(s.split("\\.").reverse)	// unapplySeq
}

def md(s: String) = s match {
	case Domain("com", _*) => "business"	// 将其他变量绑定到 _*
	case Domain("org", _*) => "non-profit"
}
```

## 正则表达式模式

`scala.util.matching.Regex`提供了一个`unapplySeq`方法：

```scala
val pattern = "a(b*)(c+)".r
val s1 = "abbbcc"
val s2 = "acc"
val s3 = "abb"

def mr(s: String) = s match {
  case pattern(a, bs) => s"""two groups "$a" "$bs""""
  case pattern(a, bs, cs) => s"""three groups "$a" "$bs" "$cs""""
  case _  => "no match"
}
```

### 字符串插值器(string interpolator)

```scala
implicit class TimeStringContext (val sc : StringContext) {
  object t {
    def apply (args : Any*) : String = sc.s (args : _*)

    def unapplySeq (s : String) : Option[Seq[Int]] = {
      val regexp = """(\d{1,2}):(\d{1,2})""".r
      regexp.unapplySeq(s).map(_.map(s => s.toInt))
    }
  }
}

def isTime(s: String) = s match {
  case t"$hours:$minutes" => Time(hours, minutes)
  case _ => "Not a time!"
}
```

## 类型匹配

```scala
def print[A](xs: List[A]) = xs match {
  case _: List[String] => "list of strings"
  case _: List[Int] => "list of ints"
}
```

```scala
import scala.reflect._
def print[A: ClassTag](xs: List[A]) = classTag[A].runtimeClass match {
  case c if c == classOf[String] => "List of strings"
  case c if c == classOf[Int]    => "List of ints"
}
```

```scala
def t(x:Any) = x match {
  case _ : Int => "Integer"
  case _ : String => "String"
}
```

## 多重匹配

```scala
def alt(x:Any) = x match {
  case 1 | 2 | 3 | 4 | 5 | 6 => "little"
  case 100 | 200 => "big"
}
```

## 联合类型

```scala
def talt(x:Any) = x match {
  case stringOrInt @ (_ : Int | _ : String) => 
    s"Union String | Int: $stringOrInt"
  case _ => "unknown"
}
```

## @switch

用于检查匹配语句能否被编译为一个`tableswitch`或`lookupswitch`的跳转表，如果被编译成一系列连续的条件语句，将会报错。

```scala
import annotation._ 

def wsw(x: Any): String = (x: @switch) match {
  case 8 => "Yes"
  case 9 => "No"
  case 10 => "No"
}

def wosw(x: Int): String = x match {
  case 8 => "Yes"
  case 9 => "No"
  case 10 => "No"
}
```

