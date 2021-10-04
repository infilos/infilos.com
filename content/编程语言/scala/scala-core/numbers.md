---
type: docs
title: "Numbers"
linkTitle: "Numbers"
weight: 16
---

## 介绍

Scala 中所有的数字类型都是对象，包含 Byte, Char, Double, Float, Int, Long, Short。这 7 种数字类型都继承自 AnyVal。同时，另外的 Unit 和 Boolean 作为非数字类型。

如果需要更复杂的数字类型，可以使用 Spire 库或 ScalaLab 库。

如果需要使用时间库，可以使用 Joda 或对 Joda 的封装 nscala-time。

## 将 String 转换为 Int

```scala
"100".toInt
"100".toDouble
"100".toFloat
"1".toLong
"1".toShort
"1".toByte
```

如果将一个实际上并不能转换为数字的字符串进行转换，会抛出 NumberFormatException 错误。

如果需要在转换时使用一个基数，即转换到对应的进制：

```scala
Inter.parseInt("1",2)
```

或者可以创建一个隐式类和对应的方法来是转换更易于使用：

```scala
implicit class StringToInt(s:String){
  def toInt(radix:Int) = Inter.parseInt(s, radix)
}

"1".toInt(2)
```

## 数字类型之间进行转换

```scala
19.45.toInt
19.toFloat
19.toLong
```

使用`isValid`方法可以检查一个数字能否转换到另一种类型：

```scala
1000L.isValidByte	// false
1000L.isValidShort	// true
```

## 覆写默认的数字类型

在向一个变量赋值时，Scala 会自动设置数字类型，可以通过几种不同的方式来设置需要的类型：

```scala
val a = 1	// Int
val a = 1d	// Double
val a = 1f	// Float
val 1 = 1L	// Long

val a = 0:Byte	// Byte

val a:Long = 1	// Long

var b:Short = _	// 设置为默认值，并不推荐的用法
val name = null.asInstanceOf[String]
```

## 使用 ++ 和 — 使数字自增或自减

这种`++`和`—`的用法在 Scala 中并不支持。因为 val 对象是不可变的，而 var 对象能够使用`+=`和`-=`来实现。并且，这些操作符是以方法的形式实现的。

## 比较浮点型数字

可以创建一个方法来设置对比浮点数需要的精度：

```scala
def ~=(x:Doubele, y:Double, precision:Double) = {
  if ((x - y).abs < precision) true else false
}

~=(0.3, 0.33333, 0.0001)
```

这个功能的应用场景在于：

```scala
0.1 + 0.2 = 0.30000000004		// 并不是等于 0.3
```

## 处理大数字

可以使用 BigInt 和 BigDecimal 来处理大的整数和浮点数。

## 生成随机数字

```scala
val r = scala.util.Random
r.nextInt
r.nextInt(100)		// 0(包含0) 到 100(不包含100) 之间
r.nextFloat
r.nextDouble
r.nextPrintableChar	// 	H
```

## 创建集合与 Range

```scala
1 to 10			// Range(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
1 to 10 by 2	// Range(1, 3, 5, 7, 9)
for (i <- 1 to 5) println(i)
for (i <- 1 until 5) println(i)

1 to 10 toArray
(1 to 10).toList
1 to 10 toList
(1 to 10).toArray
```

## 格式化输出浮点数

```scala
val pi = scala.math.Pi		// pi: Double = 3.141592653589793
println(f"$pi%1.5f")		// 3.14159
f"$pi%1.5f"					// 3.14159
f"$pi%1.2f"					// 3.14
f"$pi%06.2f"				// 003.14
```

