---
type: docs
title: "Implicits-进阶"
linkTitle: "Implicits-进阶"
weight: 9
---

隐式转换的用途：

- DSL
- Type evidence
- 减少代码冗余
- Type class
- 编译期 DI
- 扩展现有库

需要注意的地方：

- 解决规则会变得困难
- 自动转换
- 不要过度使用

## 隐式转换

一个普通的方法：

```scala
def makeString(i:Int):String = s"makeString: ${i.toString}"
```

以正常方式调用：

```scala
makeString(100)
```

但是如果我们有一个其他非 Int 类型的对象需要作为该方法的参数，则需要提供一个隐式转换，将其他类型转换为需要的 Int 类型：

```scala
case class Balance(amount:Int)
val balance = Balance(100)

implicit def balance2Int(balance:Balance):Int = balance.amount
makeString(balance)
```

### JavaConversion & JavaConverters

在与 Java 互操作时，比如我们的方法需要一个 Scala 类型的集合，而原有代码只能提供 Java 类型的集合，这时可以引入 Scala 中预定义的两种类型的隐式转换：

```scala
val javaList:java.util.List[Int] = java.util.Array.asList(1,2,3)
def someDefUsingSeq(seq:Seq[Int]) = println(seq)
import scala.collection.JavaConversions._
someDefUsingSeq(javaList)
```

或者以更好的方式，在转换时显示指定：

```scala
import scala.collection.JavaConverters
someDefUsingSeq(javaList.asScala)
```

### 隐式视图

如果我们需要包装或扩展一个已有的类型对象：

```scala
class StringWrapper(s:String){
  def quoted = s"$s"
}
```

可以以下面的方式调用：

```scala
new StringWrapper("string").quoted
```

但是并不能以下面的方式调用：

```scala
"string".quoted
```

这时，可以定义一个从`String`到`StringWrapper`的**视图**作为隐式转换：

```scala
implicit def warpString(s:String):StringWrapper = new StringWrapper(s)
"string".quoted		// "string" 会通过隐式转换自动转换为一个 StringWrapper 对象
```

**视图绑定**：废弃。

## 隐式参数

声明一个带有隐式参数的方法：

```scala
def giveMeAnInt(implicit i: Int) = i
```

可以以正常的方式调用该方法：

```scala
giveMeAnInt(1)
```

以隐式的方式提供参数：

```scala
implicit val someInt = 100
giveMeAnInt	// 100
```

但是如果同时提供多个隐式 Int 类型值，则会报错。

## 隐式类

假如其他的库已经定义了一个类：

```scala
case class Balance(amount:Int)
val balacne = 100
```

如果这时我们想对它做些操作，比如：

```scala
-balance
```

这时，我们可以创建一个包装类来扩展它：

```scala
implicit class RichBalance(val balance:Balance){
  def unary-: Balance = balance.copy(amount = -balance.amount)
}
```

这种用法主要用于扩展其他已有的库或类型。

## 隐式声明

## 作用域

查找优先级：

- 通过名字，不使用任何前缀，当前作用域
- 在隐式作用域中：
  - 伴生对象
  - 包对象
    - 源类型的包对象
    - 参数和超类、超特质的包对象

## Type class