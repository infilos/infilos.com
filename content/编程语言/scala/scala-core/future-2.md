---
type: docs
title: "Future-用例"
linkTitle: "Future-用例"
weight: 5
---

## 在 Akka 中实现事务

因为 Akka 中的`ask`模式有超时问题，这种方式不易于多重逻辑处理与 Debug，因此可以使`send`模式与`Promise`组合的方式来实现与其他 actor 的通信：在当前 actor 中创建`Promise`，通过`send`发送给其他 actor 引用，在其他 actor 中完成对`Promise`的填充，然后在当前 actor 中来处理这个被填充后的`Promise` - 即处理一个`Future`。

如果需要同时与多个 actor 通信，拿到所有 actor 的结果 - 即多个由`Promise`填充后生成的`Future`，才能完成后续的逻辑处理 - 即事务部分。只需要通过`for`表达式的方式来实现“所有需要的`Future`“都已完成。但是多个`Future`中任何一部分都会引发异常，包括事务部分，因此在最后的结果失败处理(事务回调)中要对所有的错误情况进行处理，比如通知其他的各个 actor 将刚才的操作分别进行回滚(比如买了一个东西，事务失败，重新将这个东西放回库存中，或者同时，将用户账户的余额扣款取消)。

```scala
import scala.concurrent.{ Future, Promise }
import scala.concurrent.ExecutionContext.Implicits.global

val fundsPromise:Promise[Funds] = Promise[Funds]
val sharesPromise:Promise[Shares] = Promise[Shares]

buyerActor ! GetFunds(amount, fundsPromise)
sellerActor ! GetShares(numShares, stock, sharesPromise)

val futureFunds = fundsPromise.future
val futureShares = sharesPromise.future

def transact(funds:Funds, shares:Shares):ResultType = {
  // 一些事务操作，比如更新数据库、缓存等
  // 这里也可能引发一些异常
}

val purchase = for{
  funds <- futureFunds
  shares <- futureShares
  // if ... 一些条件等等
} yield transact(funds, shares)

// 通过回调来处理事务结果
purchase onComplete{
  case Success(transcationResult) =>
  	buyerActor ! PutShares(numShares)
  	sellerActor ! PutFunds(amount)
  	// 通知其他系统事务执行成功
  case Failure(err) =>
  	// 分别检查各个操作是否成功，成功则通知其进行对应的回滚操作
  	futureFunds.onSuccess{ case _ =>  buyerActor ! PutFunds(amount) }
  	futureShares onSuccess { case _ => sellerActor ! PutShares(numShares) }
  	// 通知其他系统事务执行失败
}
```

在处理事务结果部分，如果需要得到值而不是以通知(副作用)的方式，并对事务结果进行检查以执行其他操作(回滚等)，可以使用`andThen`方法，而不是通过`onComplete`对调：

```scala
val purchaesResult:Future[Result] = purchase andThen{
  case Success(res) => ???
  case Failure(ex) => ???
}
// 然后再响应给客户端或其他后续的处理，而不需要在 onComplete 中编写大量嵌套很深的逻辑
doSomething(purchaesResult)
```

> 另一种解决方案是创建一个临时的 actor 来保存状态。

