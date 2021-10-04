---
type: docs
title: "Map Flatmap"
linkTitle: "Map Flatmap"
weight: 35
---

[Map, Map and flatMap in Scala](http://www.brunton-spall.co.uk/post/2011/12/02/map-map-and-flatmap-in-scala/)的翻译整理,点击查看原文.

## map

`map`操作会将集合中的每个元素作用到一个函数上:

	scala> val l = List(1,2,3,4,5)
	
	scala> l.map( x => x*2 )
	res60: List[Int] = List(2, 4, 6, 8, 10)

或者有些场景,你想让这个函数返回一个序列或列表,或者一个`Option`:

	scala> def f(x: Int):Option[Int] = if (x > 2) Some(x) else None
	
	scala> l.map(x => f(x))
	res63: List[Option[Int]] = List(None, None, Some(3), Some(4), Some(5))
	
## flatMap

`flatMap`的作用是,将一个函数作用的到列表中每个序列的各个元素上,**注意这里是一个嵌套的序列**,然后将这些元素展开到原始的列表中.使用一个实例来解释会比较清晰:

	scala> def g(v:Int) = List(v-1, v, v+1)
	g: (v: Int)List[Int]
	
	scala> l.map(x => g(x))
	res64: List[List[Int]] = List(List(0, 1, 2), List(1, 2, 3), List(2, 3, 4), List(3, 4, 5), List(4, 5, 6))
	
	scala> l.flatMap(x => g(x))
	res65: List[Int] = List(0, 1, 2, 1, 2, 3, 2, 3, 4, 3, 4, 5, 4, 5, 6)

这种操作对于处理`Option`类型的元素非常方便,因为`Option`同样是一种序列,只是可能包含一个元素或不包含元素:

	scala> l.map(x => f(x))
	res66: List[Option[Int]] = List(None, None, Some(3), Some(4), Some(5))
	
	scala> l.flatMap(x => f(x))
	res67: List[Int] = List(3, 4, 5)

## 使用 map 处理 Map

让我们看一下这些概念如何作用到`Map`类型上.一个`Map`可以通过多种方式实现,事实上他是一个包含二元组键值对的序列,这个二元组的第一个值是键,第二个值是值.

	scala> val m = Map(1 -> 2, 2 -> 4, 3 -> 6)
	m: scala.collection.immutable.Map[Int,Int] = Map(1 -> 2, 2 -> 4, 3 -> 6)
	
	scala> m.toList
	res69: List[(Int, Int)] = List((1,2), (2,4), (3,6))

然后通过`_1`和`_2`来方位元组的值:

	scala> val t = (1,2)
	t: (Int, Int) = (1,2)
	
	scala> t._1
	res70: Int = 1
	
	scala> t._2
	res71: Int = 2

这时如果我们要使用 map 和 flatMap 来操作`Map`,但是 map 操作在这看起来会没有意义,因为我们不会想去将我们的函数作用到一个元组,而是要将其作用到该元组的值.不过 map 提供了一种方式来处理`Map`的值,但是不包括对 key 的处理:

	scala> m.mapValues(v => v*2)
	res73: scala.collection.immutable.Map[Int,Int] = Map(1 -> 4, 2 -> 8, 3 -> 12)
	
	scala> m.mapValues(v => f(v))
	res74: scala.collection.immutable.Map[Int,Option[Int]] = Map(1 -> None, 2 -> Some(4), 3 -> Some(6))
	
## 使用 flatMap 处理 Map

但是在我的需求中我想要的处理效果更类似于 flatMap. flatMap 与 mapValues 处理`Map`的方式不同,它获取传入的元组,如果返回一个单项的`List`它会返回一个`List`; 如果返回一个元组,它会返回一个`Map`:

	scala> m.flatMap(e => List(e._2))
	res85: scala.collection.immutable.Iterable[Int] = List(2, 4, 6)
	
	scala> m.flatMap(e => List(e))
	res86: scala.collection.immutable.Map[Int,Int] = Map(1 -> 2, 2 -> 4, 3 -> 6)

这样我们可以很漂亮的使用 flatMap 来处理`Option`,我需要过滤出所有的`None`,如果仅仅使用`e => f(e._2)`会得到所有不为`None`的值并组成一个`List`返回.但是我需要的是一个`Option[Tuple2]`,如下所示:

	scala> def h(k:Int, v:Int) = if (v > 2) Some(k->v) else None
	h: (k: Int, v: Int)Option[(Int, Int)]

然后调用这个函数:

	scala> m.flatMap ( e => h(e._1,e._2) )
	res109: scala.collection.immutable.Map[Int,Int] = Map(2 -> 4, 3 -> 6)

这已经达到了我的要求,但是这些`e._1,e._2`并不是很优雅,如果有一个更好的方式将元组解包为变量就再好不过了.如果按 Python 的方式,或许在 Scala中也能工作:

	scala> m.flatMap ( (k,v) => h(k,v) )
	:10: error: wrong number of parameters; expected = 1

报错了,这并不符合预期.原因是`unapply`只能够在`PartialFunction`中执行,在 Scala中就是一个 case 语句,即:

	scala> m.flatMap { case (k,v) => h(k,v) }
	res108: scala.collection.immutable.Map[Int,Int] = Map(2 -> 4, 3 -> 6)
	
注意这里使用了大括号而不再是小括号了,表示这里是一个函数块而不是参数,而这个函数块是一个 case 语句.这意味着我们传给 flatMap 的函数块是一个`partialFunction`,并且只有与这个 case 语句匹配时才会调用,同时 case 语句中元组的`unapply`方法被调用,以将元组的值解析为变量.

## 其他方式

当然除了使用 flatMap 还有别的方式. 因为我们的目的是移除`Map`中所有不满足断言的元素,这里同样可以使用`filter`方法:

	scala> m.filter( e => f(e._2) != None )
	res114: scala.collection.immutable.Map[Int,Int] = Map(2 -> 4, 3 -> 6)
	
	scala> m.filter { case (k,v) => f(v) != None }
	res115: scala.collection.immutable.Map[Int,Int] = Map(2 -> 4, 3 -> 6)
	
	scala> m.filter { case (k,v) => f(v).isDefined }
	res116: scala.collection.immutable.Map[Int,Int] = Map(2 -> 4, 3 -> 6)

