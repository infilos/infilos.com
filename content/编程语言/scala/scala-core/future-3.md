---
type: docs
title: "Future-集合"
linkTitle: "Future-集合"
weight: 6
---

**主要解决的问题：**处理`Future`的集合，使用分组方式避免一次将过多的`Future`压入执行器队列。有效处理，每个单独`Future`可能引发的异常，同时不会丢弃`Future`的结果，对所有的`Future`结果进行累积并返回。最后抽象为一种清晰易复用的模式提供使用。

>  Scala、Akka 中的`Future`并不是 lazy 的，一旦构造会立即执行。而`scalaz`中为 lazy。Lazy 性质的`Future`实际是创建一个**执行计划**，并被最终的调用者执行。这种技术有很多优势，而本文基于标准的`Future`。

示例需要的所有引入：

```scala
import scala.language.higherKinds
import scala.collection.generic._
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global
```

## 多次调用返回 Future 的方法

首先是一个返回`Future`的方法：

```scala
trait MyService {
  def doSomething(i: Int) : Future[Unit]
}
class MyServiceImpl extends MyService {
 def doSomething(i: Int) : Future[Unit] = Future { Thread.sleep(500);println(i) }
}
```

有些场景下需要调用多次：

```scala
val svc : MyService = new MyServiceImpl
val someInts : List[Int] = (1 to 20).toList
val result : Unit = someInts.foreach(svc.doSomething _)
```

然而这会创建一个`List[Future[Unit]]`，其中包含 20 个会**立即执行**的`Future`，20 个还好，如果是成百上千个则难以承受。

程序内部，执行器会将`Future`存入队列，并使用所有可用的 worker 来执行尽可能多的`Future`。一次将过多的`Future`压入队列将会使其他需要执行器的代码进入**饥饿状态**并引起内存溢出错误。

另一方面，所有`svc.doSomething`的返回值被丢弃并赋予一个`Unit`。这里不但没有正确的等待`Future`完成，而且还把`Future`分配为`Unit`，这会扔掉所有可能的异常。

## 不能将 Future 指派为 Unit

为了避免将`Future`指派为`Unit`，可以使用`map`来替换`foreach`。同时，需要一种方式来等待所有的`Future`完成。

```scala
val futResult:Future[List[Unit]] = 
  Future.sequence{											// 2
    someInts.map(svc.doSomething _)							// 1
  }
val result: Unit = Await.result(futResult, Duration.Inf)	// 3
```

1. 使用`map`将不会丢弃`Future`的结果
2. 使用`Future.sequence`将`List[Future[Unit]]`转换为`Future[List[Unit]]`
3. 恰当的等待所有`Future`完成，这时可以安全的丢弃`List[Unit]`，因为没有抛出任何异常

当`svc.doSomething`调用过程中抛出异常时会通过`Await.result`体现。

`Future.sequence`会等待所有内部的`Future`完成，一旦完成，外部的`Future`则会完成。

> Future.sequence 源码

## 控制 Future 的执行流程

```scala
val result:Unit = 
  someInts
  	.grouped(3)										// 1
  	.toList
  	.map{ group =>
  	  val innerFutResult: Future[List[Unit]] = 		// 2
  	    Future.sequence {
  		  group.map(svc.doSomething _)
		}
		Await.result(innerFutResult, Duration.Inf)	// 3
    }.flatten										// 4
```

1. 将`someInts`每 3 个分成一组
2. 每一组创建一个`Future[List[Unit]]`
3. 使用`Await.result`等待每一个内部分组完成
4. 因为将`someInts`分割成了多个小组，这时需需要使用`flatten`将整个嵌套的分组展开

这样可以很好的解决一次将大量的`Future`压入执行器队列，同时使用`map`也不会丢弃`Future`的结果。但是有个问题就是这种方式不会返回一个`Future`结果，因为`Await.result`的使用使整个执行变成了部分同步阻塞。在实际的异步编程中，这样是不可取的。

## 返回 Future

为了使结果为`Future`，下面是新的实现：

```scala
val futResult: Future[List[Unit]] = 
  someInts
    .grouped(3)
    .toList
    .foldLeft(Future.successful(List[Unit]())){ (futAccumulator, group) =>	// 1
  	  futAccumulator.flatMap{ accumulator =>								// 2
  		val futInnerResult:Future[List[Unit]] = 
  		  Future.sequence {
  			group.map(svc.doSomething _)
		  }
		futInnerResult.map(innerResult => accumulator ::: innerResult)		// 3
	  }  
	}
val result: Unit = Await.result(futResult, Duration.Inf)
```

1. 使用`foldLeft`替换`map`，这样可以确保一次只处理一个组，然后当每个组完成时，从左到右对每个组进行累积。累计器被初始化为一个已完成的`Future.successful(List[Unit]())`。
2. 使用`Future.flatMap`替换`Future.map`，这里用于展开返回结果的类型为`Future[List[Unit]]`。如果使用`map`，返回结果将会是`Future[Future[List[Unit]]]`。
3. 一旦一个分组完成，将结果进行累积。

现在返回结果已经是一个`Future`了，但是这种使用模式很常见，但上面的写法难以复用，因此需要简化其复杂性。

### 简化

这里将会使用 for 表达式，并使用值类(value class)来实现`pimp-my-library`模式，`pimp-my-library`模式将会创建一个隐式包装器类，将方法添加到已有的类上，本质上以面向对象的方式调用新的方法。

```scala
implicit class Future_PimpMyFuture[T](val self: Future[T]) extends AnyVal {
    def get : T = Await.result(self, Duration.Inf)
  }
implicit class Future_PimpMyTraversableOnceOfFutures[A, M[AA] <: TraversableOnce[AA]](val self: M[Future[A]]) extends AnyVal {
    /** @return a Future of M[A] completes once all futures have completed */
    def sequence(implicit cbf: CanBuildFrom[M[Future[A]], A, M[A]], ec: ExecutionContext) : Future[M[A]] =
      Future.sequence(self)
  }
```

然后以下面的方式使用：

```scala
val futResult : Future[List[Unit]] =
    someInts
      .grouped(3)
      .toList
      .foldLeft(Future.successful(List[Unit]())) { (futAccumulator,group) =>
        for { 														// 1
          accumulator <- futAccumulator
          innerResult <- group.map(svc.doSomething _).sequence 		// 2
        } yield accumulator ::: innerResult
      }
 
val result : Unit = futResult.get 									// 3
```

1. 将`Future.flatMap`和嵌套的`Future.map`替换为更清晰易懂的 for 表达式
2. 使用上面预定义的“语法糖方法”替换`Future.sequence`
3. 使用上面预定义的“语法糖方法”替换`Await.result`

### 再次简化

在一个新的值类`Future_PimpMyTraversableOnce`创建另一个` pimp-my-library`方法。

```scala
implicit class Future_PimpMyTraversableOnce[A, M[AA] <: TraversableOnce[AA]](val self: M[A]) extends AnyVal {
    /** @return a Future of M[B] that completes once all futures have completed */
    def mapAsync[B](groupSize: Int)(f: A => Future[B])(implicit
      cbf: CanBuildFrom[M[Future[A]], A, M[A]],
      cbf2: CanBuildFrom[Nothing, B, M[B]],
      ec: ExecutionContext) : Future[M[B]] = {
      self
       .toList 		// 1
       .grouped(groupSize)
       .foldLeft(Future.successful(List[B]())) { (futAccumulator,group) =>
         for {
           accumulator <- futAccumulator
           innerResult <- group.map(f).sequence
         } yield accumulator ::: innerResult
       }
       .map(_.to[M]) // 2
    }
  }
```

1. 转换为`List`以有效的进行结果累积
2. 转换为预期的集合

然后使用：

```scala
val futResult : Future[List[Unit]] = someInts.mapAsync(3)(svc.doSomething _)
val result : Unit = futResult.get
```

