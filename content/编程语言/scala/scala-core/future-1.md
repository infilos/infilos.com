---
type: docs
title: "Future-基础"
linkTitle: "Future-基础"
weight: 4
---

## 介绍：Java 与 Scala 中的并发

Java 通过内存共享和锁来提供并发支持。Scala 中通过不可变状态的转换来实现：Future。虽然 Java 中也提供了 Future，但与 Scala 中的不同。

二者都是通过异步计算来表示结果，但是 Java 中需要使用阻塞的`get`方法来访问结果，同时可以在调用`get`之前使用`isDone`来检查结果是否完成来避免阻塞，但是仍然需要等待结果完成以支持后续使用该结果的计算。

在 Scala 中，无论`Future`是否完成，都可以对他指定转换过程。每一个转换过程的结果都是一个新的`Future`，这个新的`Future`表示通过函数对原始`Future`转换后得到的结果。计算执行的线程通过一个**隐式**的*execution context(执行上下文)*来决定。以不可变状态串行转换的方式来描述异步计算，避免共享内存和锁带来的额外开销。

## 锁机制的弊端

Java 平台中，每个对象都与一个逻辑**监视器**关联，以控制多线程对数据的访问。使用这种模式时需要指定哪些数据会被多线程共享并将被访问的、控制访问的和共享数据的代码段都标记为**synchronized**。Java 运行时使用**锁**的机制来确保同一时间只有一个线程能够进入被锁保护的代码段。以此协调你能够通过多线程来访问数据。

为了兼容性，Scala 提供了 Java 的并发原语。可以在 Scala 中调用方法`wait/notify/notifyAll`，并且意义与 Java 一致。但是并不提供关键字`synchronized`，但是预定义了一个方法：

```scala
var counter = 0
synchronized {
  // 这里同时只能有一个线程
  counter = counter + 1
}
```

但是这种模式难于编写可靠的多线程应用。死锁、竟态...

## 使用 Try 处理异步中的异常

当你调用一个 Scala 方法时，它会在你*等待*返回结果时执行一个计算，如果结果是一个`Future`，它表示另一个异步化执行的计算，通常会被一个完全不同的线程执行。在`Future`上执行的操作都需要一个`excution context`来提供异步执行的策略，通常可以使用由 Scala 自身提供的全局执行上下文，在 JVM 上，它使用一个**线程池**。

引入全局执行上下文：

```scala
import scala.concurrent.ExecutionContext.Implicits.global
val future = Future { Thread.sleep(10000); 21 + 21 }
```

当一个`Future`未完成时，可以调用两个方法：

```scala
future.isComplated		// false
future.value			// Option[scala.util.Try[Int]] = None
```

完成后：

```scala
future.isComplated		// true
future.value			// Option[scala.util.Try[Int]] = Some(Success(42))
```

`value`方法返回的`Option`包含一个`Try`，成功时包含一个类型为 T 的值，失败时包含一个异常，`java.lang.Throwable`的实例。

`Try`支持在尝试异步计算前进行同步计算，同时支持一个可能包含异常的计算。

**同步**计算时可以使用**try/catch**来确保新城调用方法并捕捉、处理方法抛出的异常。但是**异步**计算中，发起计算的线程常会移动到其他任务上，然后当计算中抛出异常时，原始的线程不再能通过`catch`子句来处理异常。因此使用`Future`进行异步操作时使用`Try`来处理可能的失败并生成一个值，而不是直接抛出异常。

```scala
scala> val fut = Future { Thread.sleep(10000); 21 / 0 } 
fut: scala.concurrent.Future[Int] = ...

scala> fut.value 
res4: Option[scala.util.Try[Int]] = None

// 10s later
scala> fut.value 
res5: Option[scala.util.Try[Int]] = Some(Failure(java.lang.ArithmeticException: / by zero))
```

> `Try`的定义：
>
> ```scala
> object Try {
>   /** 通过传名参数构造一个 Try。
>    * 捕获所有 non-fatal 错误并返回一个 `Failure` 对象。
>    */
>   def apply[T](r: => T): Try[T] =
>     try Success(r) catch {				// 常规的 try、catch 调用
>       case NonFatal(e) => Failure(e)
>     }
> }
> sealed abstract class Try[+T]
> final case class Failure[+T](exception: Throwable) extends Try[T]
> final case class Success[+T](value: T) extends Try[T]
> ```

## Future 操作

### map

将传递给`map`方法的函数作用到原始`Future`的结果并生成一个新的`Future`：

```scala
val result = fut.map(x => x + 1)
```

原始`Future`和`map`转换可能在两个不同的线程上执行。

### for

因为`Future`声明了一个`flatMap`方法，因此可以使用`for`表达式来转换。

```scala
val fut1 = Future { Thread.sleep(10000); 21 + 21 }	// Future[Int]
val fut2 = Future { Thread.sleep(10000); 23 + 23 }	// Future[Int]
for { x <- fut1; y <- fut2 } yield x + y			// Future[Int]
```

因为`for`表达式是对转换的**串行化**，如果没有在`for`之前创建`Future`并不能达到并行的目的。

```scala
for { 
	x <- Future { Thread.sleep(10000); 21 + 21 } 
	y <- Future { Thread.sleep(10000); 23 + 23 } 
} yield x + y		// 需要最少 20s 的时间完成计算
```

> `for { x <- fut1; y <- fut2 } yield x + y`实际会被转化为`fut1.flatMap(x => fut2.map(y => x + y))`。
>
> `flatMap`的定义：将一个函数作用到`Future`成功时的结果并生成一个新的Future，如果原`Future`失败，新的`Future`将会包含同样的异常。

### 创建 Future

上面的例子是通过`Future`的`apply`方法来创建：

```scala
def apply[T](body: =>T)(implicit @deprecatedName('execctx) executor: ExecutionContext): Future[T] = impl.Future(body)
```

`body`是需要执行的异步计算。

创建一个**成功**的`Future`：

```scala
Future.successful { 21 + 21 }
// def successful[T](result: T): Future[T] = Promise.successful(result).future
// result 为 Future 的结果
```

创建一个**失败**的`Future`：

```scala
Future.failed(new Exception("bummer!"))
// def failed[T](exception: Throwable): Future[T] = Promise.failed(exception).future
// exception 为指定的异常
```

通过`Try`创建一个已完成的`Future`：

```scala
import scala.util.{Success,Failure}
Future.fromTry(Success { 21 + 21 })
Future.fromTry(Failure(new Exception("bummer!")))
// def fromTry[T](result: Try[T]): Future[T] = Promise.fromTry(result).future
```

常用的方式是通过`Promise`来创建，得到一个被这个`Promise`控制的`Future`，当这个`Promise`完成时对应的`Future`才会完成：

```scala
val pro = Promise[Int]			// Promise[Int]
val fut = pro.future			// Future[Int]
fut.value						// None
pro.success(42)					// 或者 pro.failure(exception)/pro.complete(result: Try[T])
fut.value						// Try[Int]] = Some(Success(42))
```

或者调用`completeWith`方法并传入一个新的`Future`，新的`Future`一旦完成则用值赋予给这个`Priomise`。

### filter & collect

`filter`用户验证`Future`的值，如果满足则保留这个值，如果不满足则会抛出一个`NoSuchElementException`异常：

```scala
val fut = Future { 42 }
val valid = fut.filter(res => res > 0)
valid.value		// Some(Success(42))
val invalid = fut.filter(res => res < 0)
invalid.value	// Some(Failure(java.util.NoSuchElementException: Future.filter predicate is not satisfied))
```

同时提供了一个`withFilter`方法，因此可以在`for`表达式中执行相同的操作：

```scala
val valid = for (res <- fut if res > 0) yield res
val invalid = for (res <- fut if res < 0) yield res
```

`collect`方法对`Future`的值进行验证并通过一个操作将其转换。如果传递给`collect`的**偏函数**符合`Future`的值，该`Future`会返回经过偏函数转换后的值，否则会抛出`NoSuchElementException`异常：

```scala
val valid = fut collect { case res if res > 0 => res + 46 }		// Some(Success(88))
val invalid = fut collect { case res if res < 0 => res + 46 }	// NoSuchElementException
```

### 错误处理：failed、fallBackTo、recover、recoverWith

#### failed

`failed`方法将一个任何类型的、错误的`Future`转换为一个成功的`Future[Throwable]`，这个`Throwable`即引起错误的异常。

```scala
val failure = Future { 42 / 0 }
failure.value			// Some(Failure(java.lang.ArithmeticException: / by zero))
val expectedFailure = failure.failed
expectedFailure.value	// Some(Success(java.lang.ArithmeticException: / by zero))
```

如果调用`failed`方法的`Future`最终是成功的，而调用`failed`方法返回的`Future`会以一个`NoSuchElementException`异常失败。因此，只有当你需要`Future`失败时，调用`failed`方法才是适当的：

```scala
val success = Future { 42 / 1 }
success.value			// Some(Success(42)), 原本是一个成功的 Future
val unexpectedSuccess = success.failed
unexpectedSuccess.value	// NoSuchElementException, 称为一个失败的 Future
```

#### fallBackTo

`fallBackTo`方法用于提供一个可替换的`Future`，以便调用该方法的`Future`失败时作为备用。

```scala
val fallback = failure.fallbackTo(success)
fallback.value
```

如果调用`fallBackTo`方法的原始`Future`执行失败，传递给`fallBackTo`的错误本质上会被忽略。但是如果调用`fallBackTo`提供的`Future`也失败了，则会返回最初的错误，即原始`Future`中的错误：

```scala
val failedFallback = failure.fallbackTo( 
	Future { val res = 42; require(res < 0); res } // 这里实际是一个 require 异常
)
failedFallback.value	// Some(Failure(java.lang.ArithmeticException: / by zero))，仍然返回了原始 Future 中的除零异常
```

#### recover

`recover`允许将一个失败的`Future`转换为一个成功的`Future`，或者原始`Future`成功时则不作处理。

```scala
val recovered = failedFallback recover { case ex: ArithmeticException => -1 }
recovered.value		// Some(Success(-1)), 捕捉异常并设置成功值，返回新的 Future
```

如果原始`Future`成功，`recover`部分会以相同的值完成：

```scala
val unrecovered = fallback recover { case ex: ArithmeticException => -1 }
unrecovered.value	// Some(Success(42))
```

同时，如果传递给`recover`的偏函数并不包含原始`Future`的错误类型，新的`Future`仍然会以原始`Future`中的失败完成：

```scala
val alsoUnrecovered = failedFallback recover { case ex: IllegalArgumentException => -2 }
alsoUnrecovered.value	// Some(Failure(java.lang.ArithmeticException: / by zero))
```

#### recoverWith

`recoverWith`与`recover`类似，但是使用的是一个`Future`值。

```scala
val alsoRecovered = failedFallback recoverWith { 
	case ex: ArithmeticException => Future { 42 + 46 } 	// 这是一个 Future
}
```

其他方面的处理则于`recover`一致。

### transform：对可能性的映射

`transfor`接收两个转换`Future`的函数：一个处理原始`Future`成功的请求，一个处理失败的情况。

```scala
val first = success.transform( 
	res => res * -1, 						// 成功
	ex => new Exception("see cause", ex) 	// 失败
)
```

**注意：**现有的`transform`并不能将一个成功的`Future`转换为一个失败的`Future`，或者反向。只能对成功时的结果进行转换或失败时的异常类型进行转换。

Scala *2.12* 版本中提供了一种替代的方式，接收`Try => Try`的函数：

```scala
val firstCase = success.transform { 		// 处理成功的 Future
	case Success(res) => Success(res * -1) 
	case Failure(ex) => Failure(new Exception("see cause", ex)) 
}

val secondCase = failure.transform { 		// 处理失败的 Future
	case Success(res) => Success(res * -1) 
	case Failure(ex) => Failure(new Exception("see cause", ex)) 
}

val nonNegative = failure.transform { 		// 将失败转换为成功
	case Success(res) => Success(res.abs + 1) 
	case Failure(_) => Success(0) 
}
```

### 组合 Future：zip、fold、reduce、sequence、traverse

#### zip

`zip`方法将两个成功的`Future`转换为一个新的`Future`，其值两个`Future`值的元组。

```scala
val zippedSuccess = success zip recovered		// scala.concurrent.Future[(Int, Int)]
zippedSuccess.value								// Some(Success((42,-1)))
```

如果其中一个失败，`zip`方法的值会以同样的异常失败：

```scala
val zippedFailure = success zip failure
zippedFailure.value		// Some(Failure(java.lang.ArithmeticException: / by zero))
```

如果两个都失败，结果值会包含最初的异常，即调用`zip`方法的那个`Future`的异常。

#### fold

> `trait TraversableOnce[+A] extends GenTraversableOnce[A]`
>
> 可以被贯穿一次或多次的集合的模板特质。它的存在主要用于消除`Iterator`和`Traversable`之间的重复代码。包含一系列抽象方法并在`Iterator`和`Traversable`..中实现，这些方法贯穿集合中的部分或全部元素并返回根据操作生成的值。

`fold`方法通过穿过一个`TraversableOnce`的`Future`集合来累积值，生成一个`Future`结果。如果集合中的所有`Future`都成功了，结果`Future`会以累积值成功。如果集合中任何一个失败，结果`Future`就会失败。如果多个`Future`失败，结果中会包含第一个失败的错误。

```scala
val fortyTwo = Future { 21 + 21 }
val fortySix = Future { 23 + 23 }
val futureNums = List(fortyTwo, fortySix)
val folded = Future.fold(futureNums)(0) { 	// (0), 提供一个累积值的初始值
	(acc, num) => acc + num 
}
folded.value								// Some(Success(88))
```

#### reduce

`reduce`方法与`fold`类似，但是不需要提供初始的默认值，它使用最初的`Future`的结果作为开始值。

```scala
val reduced = Future.reduce(futureNums) { 
	(acc, num) => acc + num 
}
reduced.value	// Some(Success(88))
```

如果给`reduce`方法传入一个空的集合，则会以`NoSuchElementException`异常失败，因为没有初始值。

#### sequence

`sequence`方法将一个`TraversableOnce`的`Future`集合转换为一个包含`TraversableOnce`值的Future。比如`List[Future[Int]] => Future[List[Int]]`:

```scala
val futureList = Future.sequence(futureNums)
futureList.value	// Some(Success(List(42, 46)))
```

#### traverse

`traverse`方法将一个包含任意元素类型的`TraversableOnce`转换为一个`TraversableOnce`的`Future`，并且这个*序列*转换为一个`TraversableOnce`值的`Future`。比如，`List[Int] => Future[List[Int]]`：

```scala
val traversed =Future.traverse(List(1, 2, 3)) { i => Future(i) }	// .Future[List[Int]]
traversed.value		// Some(Success(List(1, 2, 3)))
```

### 执行副作用：foreach、onComplete、andThen

有时需要在`Future`完成时执行一些副作用，而不是通过`Future`生成一个、一些值。

#### foreach

最基本的`foreach`方法会在`Future`**成功**完成时执行一些副作用。失败时将不会执行：

```scala
failure.foreach(ex => println(ex))		// 不会执行
success.foreach(res => println(res))	// 42
```

因为不带`yield`的`for`表达式会被重写为一个`foreach`执行，因此也可以使用`for`表达式来实现：

```scala
for (res <- failure) println(res)
for (res <- success) println(res)
```

#### onComplete

这是`Future`的一种**回调**函数，无论`Future`最终成功或失败，`onComplete`方法都会执行。它需要被传入一个`Try`：`Success`用于处理成功的情况，`Failure`用于处理失败的情况：

```scala
success onComplete { 
	case Success(res) => println(res) 
	case Failure(ex) => println(ex) 
}
```

#### andThen

`Future`并不会保证通过`onComplete`注册的回调函数的执行顺序。如果需要保证回调函数的执行顺序，可以使用`andThen`方法代替，它是`Future`的两一个回调函数。

`andThen`方法返回一个对原始`Future`映射(即与原始 Future 同样的方式成功或失败)的新`Future`，但是当回调完全执行后才会完成。**它的功能是，既不影响原始 Future 的结果，又能在原始 Future 完成时执行一些回调。**

```scala
val newFuture = success andThen { 
	case Success(res) => println(res) 
	case Failure(ex) => println(ex) 
}
42					// 在回调中打印 结果
newFuture.value		// Some(Success(42)), 同时仍然保持了原始 Future 的值
```

但是需要注意的是，如果传递给`andThen`的函数如果在执行时引发异常，该异常会传递给后续的回调或者通过结果`Future`呈现。

### 2.12 中的新方法

#### flatten

`flatten`方法将一个嵌套的`Future`转换为一个单层的`Future`，即`Future[Future[Int]] =>Future[Int] `：

```scala
val nestedFuture = Future { Future { 42 } }		// Future[Future[Int]]
val flattened = nestedFuture.flatten			// Future[Int]
```

#### zipWith

`zipWith`方法实质上是对两个`Future`执行`zip`方法，并将结果元组执行一个`map`调用：

```scala
val futNum = Future { 21 + 21 }
val futStr = Future { "ans" + "wer" }
val zipped = futNum zip futStr
val mapped = zipped map { case (num, str) => s"$num is the $str" }
```

使用`zipWith`只需要一步：

```scala
val fut = futNum.zipWith(futStr) { // Scala 2.12 
	case (num, str) => s"$num is the $str" 
}
```

#### transformWith

`transformWith`支持通过一个`Try => Future`的函数来转换`Future`：

```scala
val flipped: Future[Int] = success.transformWith { // Scala 2.12 
	case Success(res) => Future { throw new Exception(res.toString) } 
	case Failure(ex) => Future { 21 + 21 } 
}
```

该方法实质上是对`transform`方法的重写，它支持生成一个`Future`而不是生成一个`Try`。

## 测试 Future

Future 的作用在于避免**阻塞**。在很多 JVM 实现上，创建上千个线程之后，线程间的上下文切换对性能的影响达到不能接受的程度。通过避免阻塞，可以繁忙时维持有限的线程数。不过，Scala 支持在需要的时候阻塞`Future`的结果，通过`Await`。

```scala
val fut = Future { Thread.sleep(10000); 21 + 21 }
val x:Int = Await.result(fut, 15.seconds) 		// <= blocks
```

然后就可以对其结果进行测试：

```scala
import org.scalatest.Matchers._
x should be (42)
```

或者直接通过特质`ScalaFutures`提供的阻塞结构来测试。比如`futureValue`方法，它会阻塞直到`Future`完成，如果`Future`失败，则会抛出`TestFailedException`异常。

```scala
import org.scalatest.concurrent.ScalaFutures._
val fut = Future { Thread.sleep(10000); 21 + 21 }
fut.futureValue should be (42)			// <= futureValue 阻塞
```

或者使用 **ScalaTest 3.0** 提供的异步测试风格：

```scala
import org.scalatest.AsyncFunSpec 
import scala.concurrent.Future

class AddSpec extends AsyncFunSpec {

	def addSoon(addends: Int * ): Future[Int] = Future { addends.sum }

	describe("addSoon") { 
		it("will eventually compute a sum of passed Ints") { 
			val futureSum: Future[Int] = addSoon(1, 2) 
			// You can map assertions onto a Future, then return 
			// the resulting Future[Assertion] to ScalaTest: 
			futureSum map { sum => assert(sum == 3) } 
		}
	}
}
```

