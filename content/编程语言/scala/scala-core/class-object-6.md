---
type: docs
title: "整体类层级"
linkTitle: "整体类层级"
weight: 25
---

Scala 中，所有的类都继承自一个共同的超类，`Any`。因此所有定义在`Any`中的方法称为通用方法，任何对象都可以调用。并且在层级的最底层定义了`Null`和`Nothing`，作为所有类的子类。

## 类层级

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/Scala-hierarchy.jpg" style="display:block;width:50%;" alt="NAME" align=center /> </div>

可以看到，顶层类型为`Any`，下面分为两个大类，`AnyVal`包含了所有**值类**，`AnyRef`包含了所有**引用类**。

**值类**共包含 9 种：Byte, Short, Char, Int, Long, Float, Double, Boolean, 和 Unit。前 8 中与 Java 中的原始类型一致，在运行时都变现为 Java 原始类型。`Unit`等价于 Java 中的`void`类型，表示一个方法并没有返回值，而它的值只有一个，写作`()`。

**引用类**对应于 Java 中的`Object`，`AnyRef`只是`java.lang.Object`的别名，因此所有 Scala 或 Java 中编写的类都是`AnyRef`的子类。

`Any`中定义了一下多个方法：

```scala
final def ==(that: Any): Boolean 
final def !=(that: Any): Boolean 
def equals(that: Any): Boolean 
def ##: Int 
def hashCode: Int 
def toString: String
```

因此所有对象都能够调用这些方法。

## 原始类的实现方式

Scala 与 Java 以同样的方式存储整数：以 32 位数字存放。这对在 JVM 上的效率以及与 Java 库的互操作性都很重要。标准的操作如加减乘除都被实现为基本操作。

但是，当整数需要被当做 Java 对象看待时，比如在整数上调用`toString`或将整数赋值给`Any`类型的变量，这时，Scala 会使用“备份”类`java.lang.Integer`。需要的时候，`Int`类型的整数能被透明转换为`java.lang.Integer`类型的**装箱整数**。

比如一个 Java 程序：

```java
boolean isEqual(int x, int y){
  return x == y;
}
System.out.println(isEqual(1,1))		// true
```

但是如果将参数类型改为`Integer`：

```java
boolean isEqual(Integer x, Integer y){
  return x == y;
}
System.out.println(isEqual(1,1))		// false
```

在调用`isEqual`时，整数 1 会被**自动装箱**为`Integer`类型，而`Integer`为引用类型，`==`在比较引用类型时比较的是引用相等性，因此结果为`false`。

但是在 Scala 中，`==`被设计为**对类型表达透明**。对于值类来说，就是自然(数学)的相等。对于引用类型，`==`被视为继承自`Objct`的`equals`方法的别名。而这个`equals`方法最初始被定义为引用相等，但被许多子类重写实现为自然理念上(数据值)的相等。因此，在 Scala 中使用`==`来判断引用类型的相等仍然是有效的，不会落入 Java 中关于字符串比较的陷阱。

 而如果真的需要进行引用相等的比较，可以直接使用`AnyRef`类的`eq`方法，它被实现为引用相等并且不能被重写。其反义比较，即引用不相等的比较，可以使用`ne`方法。

## 底层类型

类层级的底部有两个类型，`scala.Null`和`scala.Nothing`。他们是用统一的方式来处理 Scala 面向对象类型系统的某些**边界情况**的特殊类型。

`Null`类是`null`引用对象的类型，它是每个引用类的子类。`Null`不兼容值类型，比如把`null`赋值给值类型的变量。

`Nothing`类型在 Scala 类层级的最底端，它是**任何其他类型的子类型**。并且该类型没有任何值，它的一个用处是标明一个不正常的终止，比如`scala.sys`中的`error`方法：

```scala
def error(message: String): Nothing
```

调用该方法始终会抛出异常。因为`error`方法的返回值是`Nothing`类型，我们可以简便的利用该方法：

```scala
def divide(x:Int, y:Int): Int = 
  if (y != 0) x / y
  else error("can't divide by zero!")	// 返回 Nothing，是 Int 的子类型，兼容
```

另外，空的列表`Nil`被定义为`List[Nothing]`，因为`List[+A]`是协变的，这使得`Nil`可以是任何`List[T]`实例，`T`为任意类型。

