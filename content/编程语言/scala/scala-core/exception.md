---
type: docs
title: "Exception"
linkTitle: "Exception"
weight: 32
---

## Java 中的异常

Java 中异常分为两类：受检异常、非受检异常(RuntimeException, 运行时异常)。

两种异常的处理方式：

1. 非受检异常
   1. 捕获
   2. 抛出
   3. 不处理
2. 受检异常(除了 RuntimeException 都是受检异常)
   1. 继续抛出，消极的方式，一致抛出到 JVM 来处理
   2. 使用`try..catch`块来处理

**受检异常必须处理，否则不能编译通过。**

## 异常原理

异常，即程序运行期键发生的不正常事件，它会打断指令的正常流程。异常均出现在程序运行期，编译期的问题成为语法错误。

异常的处理机制：

1. 当程序在运行过程中出现异常，JVM 会创建一个该类型的异常对象。同时把这个异常对象交给运行时系统，即抛出异常。
2. 运行时系统接收到一个异常时，它会在异常产生的代码上下文附近查找对应的处理方式。
3. 异常的处理方式有两种：
   1. 捕获并处理：在抛出异常的代码附近显式使用`try..catch`进行处理，运行时系统捕获后会查询相应的`catch`处理块，在`catch`处理块中对异常进行处理。
   2. 查看异常发生的方法是否向上声明异常，有向上声明，向上级查询处理语句，如果没有向上声明， JVM 中断程序的运行并处理，即使用`throws`向外声明

## 异常分类

所有的错误和异常均继承自`java.lang.Throwable`。

1. Error：错误，JVM 内部的严重问题，无法恢复，程序人员不用处理。
2. Exception：异常，普通的问题，通过合理的处理，程序还可以回到正常的处理流程，要求编程人员进行处理。
3. RuntimeException：非受检异常，这类异常是编程人员的逻辑问题，即程序编写 BUG。Java 编译器不强制要求处理，因此这类异常在程序中可以处理也可以不处理。比如：算术异常、除零异常等。
4. 非 RuntimeException：受检异常，这类异常由外部的偶然因素导致。Java 编译器强制要求处理，程序人员必须对这类异常进行处理。比如：Exception、FileNotFoundException、IOException等。

**除了受检异常，都是非受检异常。**

## 异常处理方式

### try/catch

```java
try{
  // 可能会出现异常的代码块
} catch(异常类型1 变量名1){
  // 对该类型异常的处理代码块
} catch(异常类型2 变量名2){
  // ...
} finally{
  // 无论是否发生异常都会执行的代码块
  // 常用来释放资源，比如关闭文件
}
```

### 向上声明

即使用`thorws`关键字，将异常向上抛出，声明一个方法可能会抛出的异常列表。

```java
... methodName(参数列表) throws 异常类型1, 异常类型2 {
  // 方法体
}
```

这种方式通过声明，告诉本方法的调用者，在使用本方法时，应该对那些异常进行处理。

### 手动抛出

当程序逻辑不符合预期，要终止后面的代码执行时使用这种方式。

在方法的代码段中，可以使用`throw`关键字手动抛出一个异常。

如果手动抛出的是一个受检异常，那么本方法必须处理(应该采用向上抛出这个异常)，如果是非受检异常，则处理是可选的。

### 自定义异常

当需要一些跟特定业务相关的异常信息类时，可以根据实际的需求，继承`Exception`来定义**受检异常**，或者继承`RuntimeException`来定义**非受检异常**。

### 最佳实践

捕获那些已知如何处理的异常，即使用`try/catch`来处理已知类型的异常。

向上抛出那些不知如何处理的异常。

减少异常处理的嵌套。

## Scala 中的异常

Scala 中定义所有的异常为**非受检异常**，即便是`SQLException`或`IOException`。

最简单的处理方式是定义一个偏函数：

```scala
val input = new BufferReader(new FileReader(file))
try{
  for(line <- Iteratro.continually(input.readLine()).takeWhile(_ != null))
      println(line) 
} catch{
  case e:IOException => errorHandler(e)
  // case SomeOtherException(e) => ???
} finally{
  imput.close()
}
```

或者使用` control.Exception`来组合需要处理的多个异常：

```scala
Exception.handling(classOf[RuntimeException], classOf[IOException]) by println apply {
  throw new IOException("foo")
}
```

更上面的最佳实践里提到的一样，不能以**通配**的方式捕获所有异常，这会捕获到类似内存溢出这样的异常：

```scala
try{	
  ...
} catch{
  case _ => ...	// Don't do this!
}
```

如果需要捕获大部分可能出现的异常，并且不是严重致命的，可以使用`NonFatal`：

```scala
try{
  ...
} catch{
  case NonFatal(e) => println(e.getMessage)
}
```

`NonFatal`意为非致命错误，不会捕获类似`VirtualMachineError`这样的虚拟机错误。

```scala
object NonFatal {
   /**
    * Returns true if the provided `Throwable` is to be considered non-fatal, or false if it is to be considered fatal
    */
   def apply(t: Throwable): Boolean = t match {
     // VirtualMachineError includes OutOfMemoryError and other fatal errors
     case _: VirtualMachineError | _: ThreadDeath | _: InterruptedException | _: LinkageError | _: ControlThrowable => false
     case _ => true
   }
  /**
   * Returns Some(t) if NonFatal(t) == true, otherwise None
   */
  def unapply(t: Throwable): Option[Throwable] = if (apply(t)) Some(t) else None
}
```

在使用`NonFatal`捕获异常时定义的偏函数`case NonFatal(e) => ???`，这类似于模式匹配的构造器模式，会调用`NonFatal`的`unapply`方法，在`unapply`中会对异常进行判断，即调用`apply`方法，如果不属于**虚拟机错误**则进行捕获，否则将不进行捕获。这是一种捕获异常的快捷方式。

通常还有一些为了逻辑处理而主动抛出的异常需要处理，比如`assert`、`require`、`assume`。

### Option

在处理异常时，一般讲结果置为一个`Option`，成功时返回`Some(t)`，失败时返回`None`。

### Either

`Either`是一个封闭抽象类，表示两种可能的类型，它只有两个终极子类，`Left`和`Rright`，因此`Either`的实例要么是一个`Left`要么是一个`Right`。

类似于`Option`，通常也可以用于异常的处理，`Option`只能表示有结果或没有结果，`Either`则可以表示有结果时结果是什么，没有结果时“结果”又是什么，比如失败时的结果是一个异常信息等等。

```scala
sealed abstract class Either[+A, +B]
final case class Left[+A, +B](a: A) extends Either[A, B]
final case class Right[+A, +B](b: B) extends Either[A, B]
```

通常，`Left`表示失败，`Right`表示成功。伴生对象中提供了多种转换方法，比如`toOption`:

```scala
// 如果是 Left 实例
def toOption = e match {
  case Left(a) => Some(a)
  case Right(_) => None
}
// 如果是 Right 实例
def toOption = e match {
  case Left(_) => None
  case Right(b) => Some(b)
}
```

一个使用的实例：

```scala
case class FailResult(reason:String)

def parse(input:String) : Either[FailResult, String] = {
  val r = new StringTokenizer(input)
  if (r.countTokens() == 1) {
    Right(r.nextToken())
  } else {
    Left(FailResult("Could not parse string: " + input))
  }
}
```

这时如果只想要处理成功结果：

```scala
val rightFoo = for (outputFoo <- parse(input).right) yield outputFoo
```

或者使用`fold`：

```scala
parse(input).fold(
  error => errorHandler(error),
  success => { ... }
)
```

或者模式匹配：

```scala
parse(input) match {
  case Left(le) => ???
  case Riggt(ri) => ???
}
```

并不限制于用在解析或验证，也可以用在业务场景：

```scala
case class UserFault
case class UserCreatedEvent

def createUser(user:User) : Either[UserFault, UserCreatedEvent]
```

或者二选一的时候：

```scala
def whatShape(shape:Shape) : Either[Square, Circle]
```

或者与`Option`进行嵌套，返回一个异常，或者成功时包含有值或无值两种情况：

```scala
def lookup() : Either[FooException,Option[Foo]]
```

这种方式比较冗余，可以直接返回一个异常或结果：

```scala
def modify(inputFoo:Foo) : Either[FooException,Foo]
```

**不要在 Either 中返回异常，而是创建一个 case 类来表示异常的结果。**比如：

```scala
Either[FailResult,Foo]
```

### Try

`Try`与`Either`类似，但它不像`Either`将一些结果类包装在`Left`或`Right`中，它会直接返回一个`Failure[Throwable]`或`Succese[T]`。它是`try/catch`的一种简写方式，内部仍然是对`NonFatal`的处理。

它实现了`flatMap`方法，因此可以使用下面的方式，任何一个`Try`失败都会返回`Failure`：

```scala
val sumTry = for {
  int1 <- Try(Integer.parseInt("1"))
  int2 <- Try(Integer.parseInt("2"))
} yield {
  int1 + int2
}
```

或者通过模式匹配的方式对`Try`的结果进行处理：

```scala
sumTry match {
  case Failure(thrown) => Console.println("Failure: " + thrown)
  case Success(s) => Console.println(s)
}
```

或者获取失败时的异常值：

```scala
if (sumTry.isFailure) {
  val thrown = sumTry.failed.get
}
```

如果是成功的结果，`get`方法会返回对应的值。

可以使用`recover`方法处理多个`Try`链接中任意位置的异常：

```scala
val sum = for {
  int1 <- Try(Integer.parseInt("one"))
  int2 <- Try(Integer.parseInt("two"))
} yield {
  int1 + int2
} recover {
  case e => 0
}
// or
val sum = for {
  int1 <- Try(Integer.parseInt("one")).recover { case e => 0 }
  int2 <- Try(Integer.parseInt("two"))
} yield {
  int1 + int2
}
```

使用`toOption`方法将`Try[T]`转换为一个`Option[T]`。

或者与`Either`混合使用：

```scala
val either : Either[String, Int] = Try(Integer.parseInt("1")).transform(
  { i => Success(Right(i)) }, { e => Success(Left("FAIL")) }
).get
Console.println("either is " + either.fold(l => l, r => r))
```

> 将方法的返回值声明为 Try 可以告诉调用者该方法可能会抛出异常，可以达到受检异常的效果，即调用者必须要处理对应的异常，因此可以使代码更安全。虽然使用常规的 try/catch 也可以做到，但是这样更清晰。

#### 与 Future 组合使用

> Try 的存在意义就是为了用于 Future，参考 Future 对应的整理记录。

使用`Future`包装阻塞的`Try`代码块：

```scala
def blockMethod(x: Int): Try[Int] = Try {
    // Some long operation to get an Int from network or IO
    Thread.sleep(10000)
    100
  }

def tryToFuture[A](t: => Try[A]): Future[A] = {
    future {
      t
    }.flatMap {
      case Success(s) => Future.successful(s)
      case Failure(fail) => Future.failed(fail)
    }
  }

// Initiate long operation
val f = tryToFuture(blockMethod(1))
```

或者如果经常需要将`Future`与`Try`进行链接：

```scala
object FutureTryHelpers{
  implicit def tryToFuture[T](t:Try[T]):Future[T] = {
    t match{
      case Success(s) => Future.successful(s)
      case Failure(ex) => Future.failed(ex)
    }
  }
}

def someFuture:Future[String] = ???
def processResult(value:String):Try[String] = ???

import FutureTryHelpers._
val result = for{
  a <- someFuture
  b <- processResult(a)
} yield b
result.map { /* Success Block */ } recover { /* Failure Block */ }
```

或者使用`Promse`的`fromTry`方法来构建`Future`：

```scala
implicit def tryToFuture[T](t:Try[T]):Future[T] = Promise.fromTry(t).future
```

### 用法总结

- 在纯函数代码中将异常抛出到单独的非预期错误
- 使用`Option`返回可选的值
- 使用`Either`返回预期的错误
- 返回异常时使用`Try`而不是`Either`
- 捕获非预期错误时使用`Try`而不是`trt/catch`块
- 在处理`Future`时使用`Try`
- 在公共接口暴露`Try`类似于受检异常，直接使用异常替代


## 自定义异常

```scala
case class PGDBException(message:Option[String] = None, cause:Option[Throwable] = None) extends RuntimeException(PGDBException.defaultMessage(message, cause))

object PGDBException{
    def defaultMessage(message:Option[String], cause:Option[Throwable]) = {
      (message, cause) match {
        case (Some(msg), _) => msg
        case (_, Some(thr)) => thr.toString
        case _ => null
      }
    }

    def apply(message:String) = new PGDBException(Some(message), None)

    def apply(throwable: Throwable) = new PGDBException(None,Some(throwable))
  }

// usage
throw PGDBException("Already exist.")
throw PGDBException(new Throwable("this is a throwable"))
```

