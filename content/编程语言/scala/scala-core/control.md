---
type: docs
title: "Control"
linkTitle: "Control"
weight: 18
---

## 使用 for 与 foreach 循环

如果需要迭代集合中的元素，或者操作集合中的每个元素，或者是通过已有的集合创建新集合，都可以使用 for 和 foreach 来处理。

```scala
for (elem <- collection) operationOn(elem)
```

或者从循环中生成值：

```scala
val newArray = for (e <- a) yield a.toUpperCase
```

生成的集合类型与输入的集合类型一致，比如一个 Array 会生成一个 Array。

或者通过一个计数器访问集合中的元素；

```scala
for (i <- 0 until a.length) println(s"$i is ${a(i)}")
```

或者使用集合提供的`zipWithIndex`方法，然后访问索引与元素：

```scala
for ((e, count) <- a.zipWithIndex) (s"$count is $e")
```

或者使用守卫了限制处理条件：

```scala
for (i <- 1 to 10 if i < 4) println(i)
```

或者处理一个 Map：

```scala
val names = Map("fname" -> "Robert", "lname" -> "Goren")
for ((k,v) <- names) println(s"key: $k, value: $v")
```

> 在使用 for/yield 组合时，实际上是在创建一个新的集合，如果只是使用 for 循环则不会创建新的集合。for/yield 的处理过程类似于 map 操作。还有一些别的方法，如：foreach、map、flatMap、collect、reduce等都能完成类似的工作，可以根据需求选择合适的方法。

另外，还有一些特殊的使用方式；

```scala
// 简单的处理
a.foreach(println)

// 使用匿名函数
a.foreach(e => println(e.toUpperCase))

// 使用多行的代码块
a.foreach{ e =>
  val s = e.toUpperCase
  println(s)
}
```

### 工作原理

for 循环的工作原理：

- 一个简单的 for 循环对整个集合的迭代转换为在该集合上 foreach 方法的调用
- 一个带有守卫的 for 循环对整个集合的迭代转换为集合上 withFilter 方法的调用后跟一个 foreach 方法调用
- 一个 for/yield 组合表达式被转换为该集合上的 map 方法调用
- 一个带有守卫的 for/yield 组合表达式被转换为该集合上的 withFilter 方法调用后跟 map 方法调用

通过命令`scalac -Xprint:parse Main.scala`可以查看 Scala 对 for 循环的具体转换过程。

## 在多个 for 循环中使用多个计数器

可以在 for 循环中同时使用多个计数器：

```scala
for {
  i <- 1 to 2
  j <- 1 to 2
} println(s"i = $i, j = $j")
```

for 循环中的`<-`标示符被引用为一个**生成器(generator)**。

## 在 for 循环中使用守卫

有多种风格可以选择：

```scala
for (i <- 1 to 10 if i % 2 == 0) println(i)

for { 
  i <- 1 to 10
  if i % 2 == 0 
} println(i)
```

或者使用传统的方式：

```scala
for (file <- files){
  if (hasSoundFileExtension(file) && !soundFileIsLong(file))
  soundFiles += file
}
```

再或者可读性更强的方式：

```scala
for {
  file <- files
  if passFilter1(file)
  if passFilter2(file)
} doSomething(file)
```

因为 for 循环会被转换为一个 foreach 的方法调用，可以直接使用 withFilter 然后调用 foreach 方法，也能达到同样的效果。

## for/yield 组合

for/yield 组合可以通过在已有的集合的没有元素上应用一个新的算法(或转换等)来生成一个新的集合，并且，新的集合类型与输入集合保持一致。

```scala
val names = Array("chris", "ed", "maurice")
val capNames = for (e <- names) yield e.capitalize
// Array(Chris, Ed, Maurice)
```

如果对每个元素的应用部分需要多行，可以使用`{}`来组织代码块：

```scala
val capNames = for (e <- names) yield {
  // multi lines
  e.capitalize
}
```

## 实现 break 和 continue

在 Scala 中并没有提供 break 和 continue 关键字，但是通过`scala.util.control.Breaks`提供了类似的功能：

```scala
import util.control.Breaks._

object BreakAndContinueDemo extends App {
  breakable{
    for (i <- 1 to 10){
      println(i)
      if (i > 4) break	// 跳出循环
	}
  }
  
  val searchMe = "peter piper picked a peck of pickled peppers"
  var numps = 0
  for (i <- 0 until seachMe.length){
    breakable{
      if (searchMe.charAt(i) != 'p'){
  		break 		// 跳出 breakable, 而外层的循环 continue
	  } else {
   		numps += 1
	  }
	}
  }
  println("Found " + numPs + " p's in the string.")
}
```

在 Breaks 源码的定义当中：

```scala
def break(): Nothing = { throw breakException }

def breakable(op: => Unit) {
  try {
	op
  } catch {
    case ex: BreakControl =>
      if (ex ne breakException) throw ex
  }
}
```

break 的调用实际是抛出一个异常，breakable 部分会对捕捉这个异常，因此也就达到了“跳出循环”的效果。

而在上面例子的第二部分，breakable 实际上是控制的 if/else 部分，当满足了 if 的条件，执行 break，否则执行 `numps += 1`，即在不能满足使`numps +=1`的条件时跳过了当前元素，从而达到 continue 效果。

### 通用语法

break：

```scala
breakable { for (x <- xs) { if (cond) break } }
```

continue:

```scala
for (x <- xs) { breakable { if (cond) break } }
```

有些场景需要处理嵌套的 break：

```scala
object LabledBreakDemo extends App {
  import scala.util.control._
  
  val Inner = new Breaks
  val Outer = new Breaks
  
  Outer.breakable{
    for (i <- 1 to 5){
      Inner.breakable{
  		for (j <- 'a' to 'e') {
  		  if (i == 1 && j == 'c') Inner.break else println(s"i: $i, j: $j")
  		  if (i == 2 && j == 'b') Outer.break
		}
	  }
	}
  }
}
```

### 其他方式

如果不想使用 break 这样的语法，还有其他的方式可是实现。

- 通过在外部设置一个标记，满足条件是设定该标记，而执行时检查该标记：

  ```scala
  var barrelIsFull = false 
  for (monkey <- monkeyCollection if !barrelIsFull){   
    addMonkeyToBarrel(monkey) 
    barrelIsFull = checkIfBarrelIsFull
  }
  ```

- 通过 return 来结束循环

  ```scala
  def sumToMax(arr: Array[Int], limit: Int): Int = { 
    var sum = 0 
    for (i <- arr) { 
      sum += i 
      if (sum > limit) return limit 
    } sum 
  } 
  val a = Array.range(0,10) 
  println(sumToMax(a, 10))
  ```

## 使用 if 实现三目运算符

```scala
val absValue = if (a < 0) -a else a
println(if (i == 0) "a" else "b")
hash = hash * prime + (if (name == null) 0 else name.hashCode)
```

> if 表达式会返回一个值。

## 使用 match 语句

```scala
val month = i match { 
  case 1 => "January" 
  case 2 => "February" 
  case 3 => "March" 
  ...
  case _ => "Invalid month"
}
```

当把 match 作为一个 switch 功能使用时，推荐的做法是使用`@switch`注解。如果当前的用法不能被编译为一个 tableswitch 或 lookupswitch 时将发出警告：

```scala
import scala.annotation.switch
class SwitchDemo {
  val i = 1 
  val x = (i: @switch) match { 
    case 1 => "One" 
    case 2 => "Two" 
    case _ => "Other" 
  }
}
```

## 在一个 case 语句中匹配多个条件

如果有些场景中，多个不同条件都属于同一个业务逻辑，这时可以在一个 case 语句中添加多个条件，使用符号`|`分割，各种条件的关系为`或`：

```scala
val i = 5 
i match { 
  case 1 | 3 | 5 | 7 | 9 => println("odd") 
  case 2 | 4 | 6 | 8 | 10 => println("even") 
}
```

## 将匹配表达式的结果赋给你个变量

匹配语句的结果可以作为一个值赋值给一个变量：

```scala
val evenOrOdd = someNumber match { 
  case 1 | 3 | 5 | 7 | 9 => println("odd") 
  case 2 | 4 | 6 | 8 | 10 => println("even") 
}
```

## 访问匹配语句中默认 case 的值

如果想要访问默认 case 的值，需要使用一个变量名将其绑定，而不能使用通配符`_`：

```scala
i match { 
  case 0 => println("1") 
  case 1 => println("2") 
  case default => println("You gave me: " + default) 
}
```

## 在匹配语句中使用模式匹配

匹配语句中可以使用多种模式，比如：常量模式、变量模式、构造器模式、序列模式、元组模式或类型模式。

```scala
def echoWhatYouGaveMe(x:Any):String = x match{
  // 常量模式
  case 0 => "zero"
  case true => "true"
  case "hello" => "hello"
  case Nil => "an empty list"
  
  // 序列模式
  case List(0,_,_) => "一个长度为3的列表，且第一个元素为0"
  case List(1,_*) => "含有多个元素的列表，且第一个元素为1"
  case Vector(1, _*) => "含有多个元素的 Vector，且第一个元素为1"
  
  // 元组模式
  case (a, b) => "匹配 2 元组模式"
  case (a,b,c) => "匹配 3 元组模式"
  
  // 构造器模式
  case Person(first, "Alexander") => "匹配一个 Person，第二个字段为 Alexander，并绑定第一个字段到变量 first 上"
  case Dog("Suka") => "匹配一个 Dog，却字段值为 Suka"
  case obj @ Some(value) => "匹配一个 Some 并取出构造器中的值，同时将整个对象绑定到变量 obj"
  
  // 类型模式
  case s:String => "String"
  case i: Int => "Int"
  ...
  case d: Dog => "匹配任何 Dog 类型，并将该对象绑定到变量 d"
  
  // 通配模式
  case _ => "匹配所有上面没有匹配到的值"
}
```

## 在匹配表达式中使用 case 类

匹配 case 类或 case 对象的多种方式，用法的选择取决于你要在 case 语句右边使用哪部分值：

```scala
trait Animal 
case class Dog(name: String) extends Animal 
case class Cat(name: String) extends Animal 
case object Woodpecker extends Animal

object CaseClassTest extends App {
  def determiType(x:Animal):String = x match {
    case Dog(moniker) => "将 name 字段的值绑定到变量 moniker"
    case _:Cat => "仅匹配所有的 Cat 类"
    case Woodpecker => "匹配 Woodpecker 对象"
    case _ => "通配"
  }
}
```

## 匹配语句中使用守卫

可以给每个单独的匹配语句添加额外的一个或多个守卫：

```scala
i match { 
  case a if 0 to 9 contains a => println("0-9 range: " + a) 
  case b if 10 to 19 contains b => println("10-19 range: " + b) 
  case c if 20 to 29 contains c => println("20-29 range: " + c) 
  case _ => println("Hmmm...") 
}
```

或者是将一个对象的不同条件分拆到多个 case 语句：

```scala
num match { 
  case x if x == 1 => println("one, a lonely number") 
  case x if (x == 2 || x == 3) => println(x) 
  case _ => println("some other value") 
}
```

## 使用匹配语句代替 isInstanceOf

如果需要匹配一个类型或多种不同的类型，虽然可以使用`isInstanceOf`来进行类型的判断，但这样并不遍历同时也不提倡这种用法：

```scala
if (x.isInstanceOf[Foo]) { do something ...
```

更好的方式是使用匹配语句：

```scala
trait SentientBeing 
trait Animal extends SentientBeing 
case class Dog(name: String) extends Animal 
case class Person(name: String, age: Int) extends SentientBeing

// later in the code ... 
def printInfo(x: SentientBeing) = x match { 
  case Person(name, age) => // handle the Person 
  case Dog(name) => // handle the Dog 
}
```

## 使用匹配语句处理 List

List 结构与其他的集合结构有点不同，它以常量单元开始，以 Nil 元素结束。

```scala
val x = List(1,2,3)
val y = 1 :: 2 :: 3 :: Nil
```

在编写递归算法时，可以利用最后一个元素为 Nil 对象的便利。比如下面的 listToString 方法，如果当前的元素不是 Nil，则继续递归调用列表剩余的部分。一旦当前元素为 Nil，停止递归调用并返回一个空字符串：

```scala
def listToString(list: List[String]): String = list match { 
  case s :: rest => s + " " + listToString(rest) 
  case Nil => "" 
}
```

可以用同样的方式来递归求所有元素之和：

```scala
def sum(list: List[Int]): Int = list match { 
  case Nil => 1 
  case n :: rest => n + sum(rest) 
}
```

或者元素之积：

```scala
def multiply(list: List[Int]): Int = list match { 
  case Nil => 1 
  case n :: rest => n * multiply(rest) 
}
```

注意，这些用法必须记得要处理 Nil 元素。

## 在 try/catch 中处理多种异常

```scala
try {
  openAndReadAFile(filename) 
} catch { 
  case e: FileNotFoundException => println("Couldn't find that file.") 
  case e: IOException => println("Had an IOException trying to read that file") 
}
```

或者并不关心异常的种类，可以使用一个高阶异常类型来捕获可能的异常：

```scala
try {
  openAndReadAFile("foo") 
} catch {
  case t: Throwable => t.printStackTrace() 
}
```

或者：

```sclaa
try {
  val i = s.toInt 
} catch {
  case _: Throwable => println("exception ignored") 
}
```

Java 中可以在 catch 部分抛出异常，但是 Scala 中没有受检异常，不必指定一个方法会抛出的异常：

```scala
def toInt(s: String): Option[Int] = try {
    Some(s.toInt) 
  } catch {
	case e: Exception => throw e 
  }
```

如果想要声明抛出的异常类型，或者要与 Java 集成，可以使用`@throws`标注方法的异常类型：

```scala
@throws(class[NumberFormatException])
def toInt(s: String): Option[Int] = try {
    Some(s.toInt) 
  } catch {
	case e: Exception => throw e 
  }
```

## 在 try/catch/finally 语句块之外声明一个变量

如果需要在 Try 语句块内使用一个变量，并且需要在最后的 finally 块中访问，比如一个资源对象需要在 finally 中关闭：

```scala
object CopyBytes extends App {

  var in = None: Option[FileInputStream] 
  var out = None: Option[FileOutputStream]

  try { 
    in = Some(new FileInputStream("/tmp/Test.class")) 
    out = Some(new FileOutputStream("/tmp/Test.class.copy")) 
    var c = 0 
    while ({c = in.get.read; c != −1}) {
      out.get.write(c) 
    }
  } catch {
	case e: IOException => e.printStackTrace 
  } finally { 
    println("entered finally ...") 
    if (in.isDefined) in.get.close 
    if (out.isDefined) out.get.close 
  }
}
```

或者使用更简洁的方式：

```scala
try { 
  in = Some(new FileInputStream("/tmp/Test.class")) 
  out = Some(new FileOutputStream("/tmp/Test.class.copy")) 
  in.foreach { inputStream => 
    out.foreach { outputStream => 
      var c = 0 
      while ({c = inputStream.read; c != −1}) { 
        outputStream.write(c)
      }
    }
  }
}
```

## 自定义控制结构

