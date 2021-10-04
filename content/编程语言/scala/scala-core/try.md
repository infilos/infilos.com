---
type: docs
title: "Try"
linkTitle: "Try"
weight: 12
---

## 介绍

类型`Try`表示一个计算，它的结果可能是一个异常，或者成功完成计算的值。它类似于类型`Either`但是语义上完全不同。

`Try[T]`的实例必须是一个`scala.util.Success[T]`或`scala.util.Failure[T]`。

一个实例：处理用户的输入并计算，并且在可能会抛出异常的位置并不需要显式的异常处理：

```scala
import scala.io.StdIn
import scala.util.{ Try, Sucess, Failure }

def divide: Try[Int] = {
  val dividend: Try[Int] = Try(StdIn.readLine("Enter an Int that you'd like to divide:\n").toInt)		// 有可能会抛出异常，比如用户输入一个不能转换为 Int 的值
  val divisor: Try[Int] = Try(StdIn.readLine("Enter an Int that you'd like to divide by:\n").toInt)			// 有可能会抛出异常
  val problem: Try[Int] = dividend.flatMap(x => divisor.map(y => x/y))	// TIP
  problem match {
    case Success(v) =>
      println("Result of " + dividend.get + "/"+ divisor.get +" is: " + v)
      Success(v)
    case Failure(e) =>
      println("You must've divided by zero or entered something that's not an Int. Try again!")
      println("Info from the exception: " + e.getMessage)
      divide		// 递归调用自身，重新获取用户输入
  }
}
```

**TIP**部分展示了`Try`的重要属性，它能够进行*管道、链式*的方式组合操作，同时进行异常的捕获。

**它只能捕获 non-fatal 异常，如果是系统致命异常，将会抛出。**查看`scala.util.control.NonFatal`。

## `sealed abstract class Try[+T]`

`Try`本身定义为一个**封闭抽象类**，继承它的有两个子类：

```scala
final case class Failure[+T](exception: Throwable) extends Try[T]
final case class Success[+T](value: T) extends Try[T]
```

因此，`Try`的实例有么是一个`Failure`，要么是一个`Success`，分别表示*失败*或*成功*。

两个子类都只有一个类成员，`Failure`为一个异常，`Success`为一个值。

### toOption

```scala
def toOption: Option[T] = if (isSuccess) Some(get) else None
```

如果是一个`Success`就返回它的值，是`Failure`则返回None。

用例：

```scala
def decode(text:String):Option[String] = Try { base64.decode(text) }.toOption
```

### transform

```scala
def transform[U](s: T => Try[U], f: Throwable => Try[U]): Try[U] =
    try this match {
      case Success(v) => s(v)
      case Failure(e) => f(e)
    } catch {
      case NonFatal(e) => Failure(e)
    }
```

接收两个偏函数，一个用于处理成功的情况，一个用于处理失败的情况。根据对应偏函数的处理结果，生成一个新的`Try`实例。

## object Try

这是`Try`的伴生对象，其中定义了`Try`的构造器：

```scala
def apply[T](r: => T): Try[T] =
    try Success(r) catch {
      case NonFatal(e) => Failure(e)
    }
```

可见，构造器中只是进行了普通的`try/catch`处理，并且对`NonFatal`异常进行捕获。成功则返回一个`Success`实例，失败则返回一个`Failure`实例。

## Failure

### isFailure & isSuccess

```scala
def isFailure: Boolean = true
def isSuccess: Boolean = false
```

覆写父类抽象方法，分别写死为`true`和`false`。用于判断是否成功。

### recoverWith

```scala
def recoverWith[U >: T](f: PartialFunction[Throwable, Try[U]]): Try[U] =
   try {
     if (f isDefinedAt exception) f(exception) else this
   } catch {
     case NonFatal(e) => Failure(e)
   }
```

接受一个`Throwable => Try[U]`类型的偏函数，如果该偏函数定义了原始`Try`抛出的异常，将异常转换为一个新的`Try`实例，否则，返回原始`Try`的异常。

最后，对偏函数的执行或原始异常进行`NonFatal`捕获，生成一个可能的新的`Failure`实例。

### get

```scala
def get: T = throw exception
```

直接抛出异常，即实例成员。

### flatMap

```scala
def flatMap[U](f: T => Try[U]): Try[U] = this.asInstanceOf[Try[U]]
```

接收一个`T => Try[U]`类型的偏函数，将本身转换为当前偏函数返回值类型。

### flatten

```scala
def flatten[U](implicit ev: T <:< Try[U]): Try[U] = this.asInstanceOf[Try[U]]
```

### foreach

```scala
def foreach[U](f: T => U): Unit = ()
```

接收一个`T => U`类型的偏函数，因为实例本身的成员是一个异常，因此对`Failure`调用只会返回一个`Unit`而不会真正对成员执行传入的偏函数。

### map

```
 def map[U](f: T => U): Try[U] = this.asInstanceOf[Try[U]]
```

接收一个`T => U`的偏函数。

### filter

```scala
def filter(p: T => Boolean): Try[T] = this
```

接收一个`T => Boolean`类型的偏函数，对实例成员进行过滤，直接返回实例本身，结果仍然是一个包含异常的`Failure`。

### recover

```scala
def recover[U >: T](rescueException: PartialFunction[Throwable, U]): Try[U] =
    try {
      if (rescueException isDefinedAt exception) {
        Try(rescueException(exception))
      } else this
    } catch {
      case NonFatal(e) => Failure(e)
    }
```

接受一个`Throwable => U`类型的偏函数，如果偏函数定义了原始异常，则通过偏函数来处理原始异常并生成一个新的`Try`实例，否则返回自身。

### failed

```scala
def failed: Try[Throwable] = Success(exception)
```

将自身转换为一个包含自身成员异常的`Success`实例。

## Success

### isFailure & isSuccess

```scala
def isFailure: Boolean = false
def isSuccess: Boolean = true
```

重写父类方法并分别写死为`false`和`true`。

### recoverWith

```scala
def recoverWith[U >: T](f: PartialFunction[Throwable, Try[U]]): Try[U] = this
```

接收一个`Throwable => Try[U]`类型的偏函数，因为自身成员并不是异常类型，因此直接返回自身实例，不做异常处理。

### get

`def get = value`，直接返回自身成员的值。

### flatMap

```scala
def flatMap[U](f: T => Try[U]): Try[U] =
    try f(value)
    catch {
      case NonFatal(e) => Failure(e)
    }
```

接收一个`T => Try[U]`类型的偏函数，将自身成员应用到该偏函数，成功则生成一个新的`Try`，否则进行异常捕获。

### flatten

```scala
def flatten[U](implicit ev: T <:< Try[U]): Try[U] = value
```

返回成员值。

### foreach

```scala
def foreach[U](f: T => U): Unit = f(value)
```

接收一个函数并将成员值作用到该函数。

### map

```scala
def map[U](f: T => U): Try[U] = Try[U](f(value))
```

接收一个函数，将成员值作用到该函数并生成一个新的`Try`。

### filter

```scala
def filter(p: T => Boolean): Try[T] = {
    try {
      if (p(value)) this
      else Failure(new NoSuchElementException("Predicate does not hold for " + value))
    } catch {
      case NonFatal(e) => Failure(e)
    }
  }
```

如果成员值满足传入的函数，则返回自身，否则，返回一个包含`NoSuchElementException`异常的`Failure`实例。

### recover

```scala
def recover[U >: T](rescueException: PartialFunction[Throwable, U]): Try[U] = this
```

返回自身，不做处理。

### failed

```scala
def failed: Try[Throwable] = Failure(new UnsupportedOperationException("Success.failed"))
```

调用该方法时，生成一个新的包含`UnsupportedOperationException`异常的`Failure`。