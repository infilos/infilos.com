---
type: docs
title: "Type-进阶"
linkTitle: "Type-进阶"
weight: 14
---

## 类型别名

```scala
type Foo = String
type IntList = List[Int]

type MyList[T] = List[T]
val MyList = List
```

为已有类型创建一个类型别名，或者同时为其半生对象创建一个别名。

### 场景：提高 API 可用性

如果你的 API 需要引入一些外部类型，比如：

```scala
import spray.http.ContentType
import org.joda.time.DateTime

final case class RiakValue(
	contentType:ContentType,
	lastModified: DateTime
)
```

当客户端在创建`RiakVlue`实例时则必须同时引入这些**外部类型**：

```scala
import spray.http.ContentType		// <= 需要引入外部类型
import org.joda.time.DateTime

val rv = RiakValue(
	ContentType.`application/json`,
	new DateTime()
)
```

为了避免这些多次重复且必要的引入，我们可以为需要的类型创建别名并组织在一起：

```scala
package com.scalapenos
package object riak{
  type ContentType = spray.http.ContentType
  val ContentType = spray.http.ContentType
  
  val MediaTypes = spray.http.MediaTypes
  
  type DateTime = org.joda.time.DateTime
}
```

然后客户端就可以这样使用：

```scala
import com.scalapenos.rika._	// 引入所有使用 RiakValue 需要的外部类型

val rv = RiakVaule(
	ContentType.`application/json`,
	new DateTime()
)
```

### 场景：简化类型签名

有时候类型签名比较难于理解，特别是一些函数作为参数时的类型签名：

```scala
def authenticate[T](auth:RequestContext => Future[Either[Rejection, T]]) = ...
```

我们可以为这种复杂类型创建别名以隐藏复杂性：

```scala
package object authentication {
  type Authectication[T] = Either[Rejection, T]
  type ContextAuthenticator[T] = RequestContext => Future[Authection[T]]
}
```

最终得到经过简化的类型签名：

```scala
def authenticate[T](auth: ContextAutuenticator[T]) = ...
```

### 场景：任何地方都可以使用类型别名

在 Scala 标准库中的 Predef 中，为大量的常用类型定义了类型别名，以简化使用：

```scala
object Predef extends LowPriorityImplicits{
  ...
  type String        = java.lang.String
  type Class[T]      = java.lang.Class[T]
  ...
  type Function[-A, +B] = Function1[A, B]
  ...
  type Map[A, +B] = immutable.Map[A, B]
  type Set[A]     = immutable.Set[A]
  val Map         = immutable.Map
  val Set         = immutable.Set
  ...
}
```

## class Tag & type Tag

## type class



