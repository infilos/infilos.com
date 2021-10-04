---
type: docs
title: "Trait-基础"
linkTitle: "Trait-基础"
weight: 10
---

## 原理

### 定义特质

```scala
trait Philosophical{
  def philosophical() {
    println("I consume memory, therefore I am!")
  }
}
```

除了使用`trait`关键字，与类的定义一样，也没有声明超类，因此只有一个默认的`AnyRef`作为超类。同时定义了一个具体的方法。

### 在类中混入特质

然后将特质混入到一个类中：

```scala
class Forg extends Philosophical {
  override def toString = "green"
}
```

可以使用`extends`或`with`混入特质，混入特质的类同时将会隐式的继承特质的超类。

从特质继承的方法可以向使用超类中的方法一样：

```scala
val forg = new Forg()
forg.philosophical()		// I consume mem...
```

### 特质也是类型

```scala
val phil: Philosophical = forg
phil.philosophical()		// I consume mem...
```

`phil`变量被声明为`Philosophical`类型，因此它可以被初始化为任何继承了`Philosophical`特质的类的实例。

### 混入多个特质

如果需要将特质混入到一个继承自超类的类里，可以使用`with`来混入特质，或者使用多个`with`混入多个特质：

```scala
class Animal
trait HasLegs
class Forg extends Animal with Philosophical with HasLegs{
  override def toString = "green"
}
```

### 重写特质方法

```scala
class Forg extends Philosophical {
  override def toString = "green"
  override def philosophical() {
    println("It ain't easy being $toString!")
  }
}
```

这时，类`Forg`的实例仍然可以赋值到`Philosophical`类型的变量，但是其方法已经被重写。

### 特质与类的不同

特质类似于带有具体方法的 Java Interface。特质能够实现类的所有功能，但是有两点差别：

- 特质没有构造器
- 无论类的所处位置，`super`调用都是静态绑定的。但是特质中是动态绑定的。因为可以同时混入多个特质，其绑定会基于混入特质的顺序。

## 胖瘦接口

特质一种主要的应用方式是更具类已有的方法自动为类添加方法。但是在 Java 的接口中，如果需要多个接口的相互继承，必须在子接口中声明所有父接口的抽象方法。而在 Scala 中，可以通过定义一个包含部分抽象方法的特质，和一个包含大量已实现方法的特质，子类在继承多个特质后只需要实现抽象方法部分，并且同时能够获得需要的所有方法。

## 实例：Ordered 特质

比如一个需要比较的对象，希望使用`>`或`<`等操作符来进行比较：

```scala
class Rational(n:Int, d:Int){
  // ...
  def < (that.Rational) = this.numer * that.denom > that.numer * this.denom
  def > (that.Rational) = that < this
  def <= (that:Rational) = (this < that) || (this == that)
  def >= (that.Rational) = (this > that) || (this == that)
}
```

一共定义了 4 中操作符，并且后续的 3 中都是基于第一个方法，这是一个胖接口的典型，如果是瘦接口的话，只会顶一个一个`<`操作符，然后其他的功能由客户端自己实现。

这些用法很常见，因此 Scala 专门提供了一个`Ordered`特质，只需要定义一个`compare`方法，然后`Ordered`会自动创建所有的比较操作。比如：

```scala
class Rational(n:Int, d:Int) extends Ordered[Rational]{
  def compare(that:Rational) = 
    (this.numer * that.denom) - (that.numer - this.denom)
}
```

这个`compare`方法通过比较两个值的差来判断大小。可以参考`Ordered`原码：

```scala
trait Ordered[A] extends Any with java.lang.Comparable[A] {
  def compare(that: A): Int
  def <  (that: A): Boolean = (this compare that) <  0
  def >  (that: A): Boolean = (this compare that) >  0
  def <= (that: A): Boolean = (this compare that) <= 0
  def >= (that: A): Boolean = (this compare that) >= 0
  def compareTo(that: A): Int = compare(that)
}
```

## 特质叠加

当类同时混入多个特质时，混入的特质会从右到左依次执行。

如果混入特质中使用了`super`，将会调用其左侧特质中的方法。

## 多重继承

## 应用场景

1. 如果行为不会被复用，使用具体类
2. 如果需要在多个不相关的类中使用，使用特质
3. 如果需要构造参数，或者在 Java 中使用这部分代码，使用抽象类。因为 Scala 中只有那些仅包含抽象成员的特质会被翻译为 Java 的接口，否则并没有具体的模拟。
4. 如果效率非常重要，使用类。大多数 Java 运行时都能让类成员的续方法调用快于接口方法调用。



