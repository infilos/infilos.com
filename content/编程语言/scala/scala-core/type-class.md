---
type: docs
title: "Type Class"
linkTitle: "Type Class"
weight: 34
---

[《Demystifying Implicits and Typeclasses in Scala》](http://www.cakesolutions.net/teamblogs/demystifying-implicits-and-typeclasses-in-scala)一文的整理翻译。

The idea of typeclasses is that you provide evidence that a class satisfies an interface。

**类型类**思想是你提供了一个**类**满足于一个**接口**的**证明**。

```scala
trait CanFoo[A] {
  def foos(x: A): String
}

case class Wrapper(wrapped: String)

object WrapperCanFoo extends CanFoo[Wrapper] {
  def foos(x: Wrapper) = x.wrapped
}
```

**类型类**思想是你提供了一个**类**(Wrapper)满足于一个**接口***(CanFoo)的**证明**(WrapperCanFoo)。

`Wrapper`不是直接的去继承一个接口，类型类让我们把类的定义和接口的实现分开。这表示，我可以为你的类实现一个接口，或者第三方可以为你的类实现我的接口，并且一切基本结束工作。

但是有一个明显的问题，如果你想把一个东西实现为`CanFoo`，你需要同时询问你的调用者类的实例和协议。

```scala
def foo[A](thing: A, evidence: CanFoo[A]) = evidence.foos(thing)
```

