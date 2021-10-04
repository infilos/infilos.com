---
type: docs
title: "Class Object: 对象"
linkTitle: "Class Object: 对象"
weight: 23
---

## 介绍

object 在 Scala 中有多种意义，可以和 Java 一样当做一个类的实例，但是 object 本身在 Scala 就是一个关键字。

## 对象映射

如果需要将一个类的实例从一种类型映射为另一种类型，比如动态的创建对象。可以使用 asInstanceOf 来实现这种需求：

```scala
val recognizer = cm.lookup("recognizer").asInstanceOf[Recognizer]
```

类似于在 Java 中：

```java
Recognizer recognizer = (Recognizer)cm.lookup("recognizer");
```

该方法定于与 Any 类因此所有对象可用。需要注意的是，这种转换可能会抛出 ClassCastException 异常。

## Scala 中与 Java 中 .class 等效的部分

如果有些 API 需要传入一个 Class，在 Java 中调用了一个 .class，但是在 Scala 中却不能工作。在 Scala 中等效的方法是 classOf 方法，定义于 Predef 对象，因此所有的类可用。

```scala
// java 
info = new DataLine.Info(TargetDataLine.class, null);
// scala
val info = new DataLine.Info(classOf[TargetDataLine], null)
```

## 测定对象的 Class

```scala
obj.getClass
```

## 使用 Object 启动应用

有两种方式启动一个应用，即作为一个程序的入口点：

- 创建一个 object 并集成 App 特质
- 创建一个 object 并实现 main 方法

```scala
object Hello extends App {
  println("Hello, world") 
}

object Hello2 { 
  def main(args: Array[String]) {
	println("Hello, world") 
  } 
}
```

两种方式中，Scala 都是以`objct`启动应用而不是一个类。

## 创建单例对象

创建单例对象即确保只有该类一个实例存在。

```scala
object CashRegister { 
  def open { println("opened") } 
  def close { println("closed") } 
}

object Main extends App { 
  CashRegister.open 
  CashRegister.close 
}
```

CashRegister 会以单例的形式存在，类似 Java 中的静态方法。常用语创建功能性方法。

或者用于创建复用的消息对象：

```scala
case object StartMessage
case object StopMessage
actorRef ! StartMessage
```

## 使用伴生对象创建静态成员

如果需要一个类拥有实例方法和静态方法，只需要在类中创建实例(非静态)方法，在伴生对象中创建静态方法。伴生对象即，以 object 关键字定义，与类名相同并与类处于相同源文件。

```scala
// 类定义
class Pizza (var crustType: String) {
  override def toString = "Crust type is " + crustType 
}

// 伴生对象
object Pizza { 
  val CRUST_TYPE_THIN = "thin" 
  val CRUST_TYPE_THICK = "thick" 
  def getFoo = "Foo" 
}
```

实现步骤：

- 在同一源文件中定义类和 object，并且拥有相同的命名
- 静态成员定义在 obejct 中
- 非静态成员定义在类中

类与伴生对象可以互相访问对方的私有成员。

## 将通用代码放到包(package)对象

如果需要创建包级别的函数、字段或其他代码，而不需要一个类或对象，只需要将代码以 package object 的形式，放到你期望可见的包中。

比如你想要`com.alvinalexander.myapp.model`能够访问你的代码，只需要在`com/alvinalexander/myapp/model`目录中创建一个`package.scala`文件并进行一下定义：

```scala
package com.alvinalexander.myapp

package object model {
  // code
}
```

## 不使用 new 关键字创建对象实例

实现步骤：

- 为类创建一个伴生对象，并以预期的构造器签名创建 apply 方法
- 直接将类创建为 case 类

```scala
class Person {
  var name: String = _ 
}

object Person { 
  def apply(name: String): Person = { 
    var p = new Person 
    p.name = name p 
  } 
}
```

可以为类创建不同的 apply 方法，类似类的辅助构造器。

## 通过 apply 实现工厂方法

为了让子类声明创建哪种类型的对象，或者把对象的创建集中在同一个位置管理，这时候需要实现一个工厂方法。

比如创建一个 Animal 工厂，根据你提供的需要创建 Dog 或 Cat 的实例：

```scala
trait Animal {
  def speak 
}

object Animal {
  private class Dog extends Animal {
	override def speak { println("woof") } 
  }

  private class Cat extends Animal {
    override def speak { println("meow") } 
  }

  // the factory method 
  def apply(s: String): Animal = { 
    if (s == "dog") new Dog 
    else new Cat 
  }
}

val cat = Animal("cat")
val dog = Animal("dog")
```

