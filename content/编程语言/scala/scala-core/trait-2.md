---
type: docs
title: "Trait-用例"
linkTitle: "Trait-用例"
weight: 11
---

## 介绍

特质(trait)类似于 Java 中的接口(interface)。类似 Java 类能够实现多个接口，Scala 类能够扩展多个特质。

## 像 Interface 一样使用 trait

在特质中，仅**声明**在需要扩展的类中需要的**方法**。

```scala
trait BaseSoundPlayer { 
  def play 
  def close 
  def pause 
  def stop 
  def speak(whatToSay: String)
}
```

然后在需要扩展的类中使用特质，仅有一个特质时使用 extends，多于一个时第一个使用 extends，后续的使用 with：

```scala
class Mp3SoundPlayer extends BaseSoundPlayer { ...
class Foo extends BaseClass with Trait1 with Trait2 { ...
class Foo extends Trait1 with Trait2 with Trait3 with Trait4 { ...
```

除非一个类被声明为抽象类(abstract)，否则需要实现特质中的所有抽象方法：

```scala
class Mp3SoundPlayer extends BaseSoundPlayer { 
  def play { // code here ... } 
  def close { // code here ... } 
  def pause { // code here ... } 
  def stop { // code here ... } 
  def resume { // code here ... } 
}
    
abstract class SimpleSoundPlayer extends BaseSoundPlayer { 
  def play { ... } 
  def close { ... } 
}
```

同时，一个特质能够扩展另一个特质：

```scala
trait Mp3BaseSoundFilePlayer extends BaseSoundFilePlayer { 
  def getBasicPlayer: BasicPlayer 
  def getBasicController: BasicController 
  def setGain(volume: Double) 
}
```

特质中除了能够声明抽象方法，还能提供方法的实现，类似 Java 中的抽象类：

```scala
abstract class Animal {
  def speak 
}

trait WaggingTail { 
  def startTail { println("tail started") } 
  def stopTail { println("tail stopped") } 
}
```

当一个类拥有多个特质时，这些特质被称为**混入(mix)**这个类。**混入**同时用于使用特质来扩展一个单独的对象：

```scala
val f = new Foo with Trait1
```

## 在特质中使用抽象或具体的字段

可以在特质中定义抽象或具体字段，以便所有扩展他们的类型都能够使用它们。没有提供初始值的字段将会是一个抽象字段，拥有初始值的字段将会是具体字段：

```scala
trait PizzaTrait { 
  var numToppings: Int 
  var size = 14 
  val maxNumToppings = 10 
}
```

类似于抽象方法，被扩展的类需要实现所有抽象字段，否则需要声明为抽象类。

在特质中定义字段时可以使用 var 或 val，如果是 val，在子类或子特质中需要使用 override 来覆写该字段，而 var 则不需要。

## 像抽象类一样使用特质

在特质中定义的方法跟普通的 Scala 方法类似，可以直接使用或重写它们。

```scala
trait Pet { 
  def speak { println("Yo") }  	// 具体方法
  def comeToMaster 				// 抽象方法
}

class Dog extends Pet { 
  // 如果不需要则不用重写特质中的具体方法
  def comeToMaster { ("I'm coming!") } 	// 实现抽象方法
}

class Cat extends Pet { 
  override def speak { ("meow") }		// 重写具体方法 
  def comeToMaster { ("That's not gonna happen.") } 
}
```

虽然 Scala 中有抽象类，但是一个类只能继承一个抽象类，但是可以同时扩展多个方法，除非如 Classes 章节中介绍的对抽象类有特殊需要，否则，使用特质户更加灵活。

## 简单的混入

需要将多个特质混入类以提供健康的设计。为了实现简单的还如，只需要在特质中定义需要的方法，然后在需要扩展的类中使用 extends 或 with 来进行混入。

下面的例子中，同时继承自一个抽象类并混入一个特质，同时实现了抽象类中的抽象方法：

```scala
trait Tail { 
  def wagTail { println("tail is wagging") } 
  def stopTail { println("tail is stopped") } 
}

abstract class Pet (var name: String) {
  def speak // abstract 
  def ownerIsHome { println("excited") } 
  def jumpForJoy { println("jumping for joy") } 
}

class Dog (name: String) extends Pet (name) with Tail { 
  def speak { println("woof") } 
  override def ownerIsHome { wagTail speak } 
}
```

## 限定哪些类可以通过继承来使用特质

如果需要对一些特质进行限制，比如，只能添加到继承了一个抽象类或扩展了一个特质的类上。

```scala
trait [TraitName] extends [SuperThing]
```

这种声明语法称为 TraitName，TraitName 只能被混入到哪些扩展了名为 SuperThing 的类型的类中，这里的 SuperThing 可以是 特质、类、抽象类。

简单的说就是，**只有继承或扩展了 SuperThing，才能混入 TraitName 这个特质**。

```scala
class StarfleetComponent 
trait StarfleetWarpCore extends StarfleetComponent 
class Starship extends StarfleetComponent with StarfleetWarpCore
```

这个例子中，StarfleetWarpCore 继承了 StarfleetComponent，而 Starship 也继承了 StarfleetComponent，因此，类 Starship 可以混入特质 StarfleetWarpCore。

## 限定一个特质只混入到一种类型的子类

可以为一个特质添加一个类型，只有该类型的子类才能混入该特质。

```scala
trait StarfleetWarpCore { 
  this: Starship => 
  // more code here ... 
}
```

这个特质只能混入到 Starship 的子类中，比如：

```scala
class Starship 
class Enterprise extends Starship with StarfleetWarpCore
```

否则将会报错：

```scala
class RomulanShip 
class Warbird extends RomulanShip with StarfleetWarpCore	// 错误
```

详细的错误信息：

```basic
error: illegal inheritance; 
self-type Warbird does not conform to StarfleetWarpCore's selftype StarfleetWarpCore with Starship 
class Warbird extends RomulanShip with StarfleetWarpCore
```

错误中提到了 **self type**，关于 self type 的描述：

> "Any concrete class that mixes in the trait must ensure that its type conforms to the trait’s self type."
>
> 任何混入特质的实现类必须确保它的类型与特质的 self type 一致。

特质同时能够限定混入它的类型需要同时扩展其他多个类型，想要混入特质 WarpCore 的类型需要同时混入 this 关键字后面的所有类型：

```scala
trait WarpCore {
  this: Starship with WarpCoreEjector with FireExtinguisher => 
}
```

## 限定特质只能混入到拥有特定方法的类型

可以使用 self type 语法的变型来限制一个特质只能混入到拥有特定方法的类型(类、抽象类、特质)：

```scala
trait WarpCore {
  this: { def ejectWarpCore(password: String): Boolean } => 
}
```

因此，如果想要混入特质 WarpCore，需要拥有更上述方法签名一致的方法才可以。

或者需要多个方法：

```scala
trait WarpCore { 
  this: { 
    def ejectWarpCore(password: String): Boolean 
    def startWarpCore: Unit 
  } => 
}
```

这个方式称为**结构化类型**。

## 将特质添加到对象实例

不同于将特质混入到实际的类，同样能够在创建对象时，将特质扩展到一个单独的对象。

```scala
class DavidBanner

trait Angry {
  println("You won't like me ...") 
}

object Test extends App {
  val hulk = new DavidBanner with Angry 
}
```

或者更为实际的用法：

```scala
trait Debugger { 
  def log(message: String) {
    // do something with message 
  } 
}

// no debugger 
val child = new Child

// debugger added as the object is created 
val problemChild = new ProblemChild with Debugger
```

## 像特质一样扩展一个 Java Interface

如果想要在 Scala 中实现一个 Java 接口，可以和使用特质一样，使用 extends 和 with 来扩展。

首先是 Java 接口的定义：

```java
public interface Animal {
  public void speak(); 
}

public interface Wagging {
  public void wag(); 
}

public interface Running {
  public void run(); 
}
```

然后在 Scala 中像使用特质一样使用他们：

```scala
class Dog extends Animal with Wagging with Running { 
  def speak { println("Woof") } 
  def wag { println("Tail is wagging!") } 
  def run { println("I'm running!") }
}
```

区别在于 Java 中的接口都没有实现行为，因此要实现接口或声明为抽象类。

