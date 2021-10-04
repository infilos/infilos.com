---
type: docs
title: "函数式-用例"
linkTitle: "函数式-用例"
weight: 3
---

## 匿名函数

匿名函数，或函数字面值，可以将它传递给一个方法，或者赋值给一个变量。

比如过滤集合中的偶数，可以将一个匿名函数传递给集合的 filter 方法：

```scala
val x = List.range(1, 10)
val evens = x.filter((i: Int) => i % 2 == 0)
```

这里的匿名函数就是`(i: Int) => i % 2 == 0`。符号`=>`可以认为是一个转换器，将集合中的每个整数 i ，通过符号右边的表达式`i % 2 == 0`转换为一个布尔值，filter 通过这个布尔值判断这个 i 的去留。

因为 Scala 可以推断类型，因此可以省略掉类型：

```scala
val evens = x.filter(i => i % 2 == 0)
```

又因为当一个参数在函数中只出现一次时使用通配符 _ 表示，因此可以省略为：

```scala
val evens = x.filter(_ % 2 == 0)
```

在更为常见的场景中：

```scala
x.foreach((i:Int) => println(i))
```

简略为：

```scala
x.foreach((i) => println(i))
```

进一步简略：

```scala
x.foreach(i => println(i))
```

再次简略：

```scala
x.foreach(println(_))
```

最终，如果由一条语句组成的函数字面量仅接受一个参数，并不需要明确的名字和指定参数，因此最终可以简略为：

```scala
x.foreach(println)
```

## 像变量一样使用函数

可以向传递一个变量一样传递函数，就像是一个 String、Int 一样。

比如需要将一个数字转换为它的2倍，向上一节一样，将`=>`当做一个转换器，这时是将一个数字转换为它的2倍，即：

```scala
(i: Int) => { i * 2 }
```

然后将这个函数字面量赋值给一个变量：

```scala
val double = (i: Int) => { i * 2 }
```

变量 double 是一个实例，就像 String、Int 的实例一样，但是它是一个函数的实例，作为一个**函数值**，然后可以像方法调用一样使用它：

```scala
double(2) 	// 4
```

或者将它传递给一个函数或方法：

```scala
list.map(double)
```

### 如何声明一个函数

已经见过的方式：

```scala
val f = (i: Int) => { i % 2 == 0 }
```

Scala 编译器能够推断出该方法返回一个布尔值，因此这里省略了函数的返回值。一个包含完整类型的声明会是这样：

```scala
val f: (Int) => Boolean = i => { i % 2 == 0 }
val f: Int => Boolean = i => { i % 2 == 0 }
val f: Int => Boolean = i => i % 2 == 0
val f: Int => Boolean = _ % 2 == 0
```

## 将函数作为方法的参数

这种需求的处理过程：

1. 定义方法，定义想要接收并作为参数的函数签名
2. 定义一个或多个对应该签名的函数
3. 将需要的函数传入方法

```scala
def executeFunction(callback:() => Unit) {
  callback() 
}
```

签名`callback:() => Unit`表示该函数不需要参数并返回一个空值。比如：

```scala
val sayHello = () => { println("Hello") }
executeFunction(sayHello)
```

这里，方法中定义的函数名并没有实际意义，仅作为方法的一个参数名，就像一个 Int 常被命名为字母 i 一样，这里可以通用定义为任何形式：

```scala
def executeFunction(f:() => Unit) {
  f() 
}
```

同时，传入的函数必须与签名完全一致，签名通用的语法为：

```scala
methodParameterName: (functionParameterType_s) => functionReturnType
```

## 更为复杂的函数

函数参数的签名为：接收一个整形数字作为参数并返回一个空值：

```scala
def exec(callback: Int => Unit) {
  callback(1) 	// 调用传入的函数
}
val plusOne = (i: Int) => { println(i+1) }
exec(plusOne)
```

或者接收更多参数的函数：

```scala
executeFunction(f:(Int, Int) => Boolean)
```

或者返回一个集合类型：

```scala
exec(f:(String, Int, Double) => Seq[String])
```

或者接收函数参数并同时接受其他类型的参数：

```scala
def executeAndPrint(f:(Int, Int) => Int, x: Int, y: Int) { 
  val result = f(x, y) 
  println(result) 
}
```

## 使用闭包

如果需要将函数作为一个参数传递，同时又需要使函数在其声明的作用域引用以存在的变量。比如下面这个例：

```scala
class Foo {
  def exec(f:String => Unit, name:String){ f(name) }
}

object ClosureExample extends App{
  var hello = "hello"
  def sayHello(name:String) = { println(s"$hello, $name") }	// 引入闭包变量 hello
  
  val foo = new otherscope.Foo
  foo.exec(sayHello, "Al")
  
  hello = "Hola"			// 修改本地变量 hello
  foo.exec(sayHello, "Lorenzo")
}

Hello, Al
Hola, Lorenzo
```

在这个例子中，函数 sayHello 在定义时除了声明了一个正式参数 name，同时引用了当前作用域的变量 hello。将 函数 sayHello 作为一个参数传入 exec 方法后，再修改本地变量 hello 的值，sayHello中仍然能够引用到改变后的 hello 的值。这里，Scala 创建了一个闭包。

这里为了简单只只是将 sayHello 传递给了 exec 方法，同样可以将其传递到很远的位置，即多层传递。但是变量并不在 Foo 的作用域或方法 exec 的作用域中，比如在 Foo 中或 exec 中再单独打印 hello 都不会编译通过。

闭包的三个要素：

- 一段代码块可以像值一样传递
- 任何拥有这段代码块的人都可以在任何时间根据需要执行它
- 这段代码块可以在创建它的上下文中引用变量

### 创建闭包

```scala
var votingAge = 18
val isOfVotingAge = (age: Int) => age >= votingAge
```

现在可以把函数 isOfVotingAge 传递给任意作用域中的函数、方法、对象，同时 votingAge 是可变的，改变它的值同时会引起 isOfVotingAge 函数中对他引用的值的改变。

### 使用其他数据结构闭包

```scala
val fruits = mutable.ArrayBuffer("apple")
val addToBasket = (s: String) => { 
  fruits += s 
  println(fruits.mkString(", ")) 
}
```

这时，将 addToBasket 函数传递到其他作用域并执行时，都能够修改 fruits 的值。

## 使用偏应用函数

可以定义一个需要多个参数的函数，提供部分参数并返回一个偏函数，它会携带已获得的参数，最终传递给他剩余需要的参数以完成执行：

```scala
val sum = (a: Int, b: Int, c: Int) => a + b + c
```

它本身需要三个参数，当只提供两个参数时，它会返回一个偏函数：

```scala
val sum = (a: Int, b: Int, c: Int) => a + b + c
// sum: (Int, Int, Int) => Int = <function3>

val f = sum(1, 2, _: Int)
// f: Int => Int = <function1>
```

最后给他提供一个参数，完成整个计算：

```scala
f(3)
// 6
```

## 创建返回函数的函数

可以定义一个返回函数的函数，将它传递给另一个函数，最终提供需要的参数并调用。

这是一个匿名函数：

```scala
(s: String) => { prefix + " " + s }
```

定义一个函数来生成这个函数：

```scala
def saySomething(prefix: String) = (s: String) => {
  prefix + " " + s 
}
```

可以将它赋值给一个变量并通过这个拥有函数值的变量来调用函数：

```scala
val sayHello = saySomething("Hello")
sayHello("Alex")
```

或者更复杂的，根据输入值的不同从而返回不同的函数：

```scala
def greeting(language: String) = (name: String) => { 
  language match { 
    case "english" => "Hello, " + name 
    case "spanish" => "Buenos dias, " + name 
  } 
}
```

## 创建偏函数

可以创建一个函数，只对所有可能的输入值的一个子集有效(称为偏函数的原因)，或者一组这样的函数，最后通过组合来完成需要的功能。

定义一个偏函数：

```scala
val divide = new PartialFunction[Int, Int] { 
  def apply(x: Int) = 42 / x 
  def isDefinedAt(x: Int) = x != 0 	// 仅对部分不等于 0 的整数有效
}

divide.isDefinedAt(1)  					// true
if (divide.isDefinedAt(1)) divide(1)	// 42
divide.isDefinedAt(0)					// false
```

或者更为常用的模式：

```scala
val divide2: PartialFunction[Int, Int] = {
  case d: Int if d != 0 => 42 / d 
}
```

**意思就是，它只能接受 Int 类型参数的一部分(d 不等于 0)进行处理并返回一个 Int 值。**

使用 case 的方式仍然能够更第一种方式一样判断它是否能够接受一个值：

```scala
divide2.isDefinedAt(0)
divide2.isDefinedAt(1)
```

同时可以使用 orElse 将多个偏函数**组合**，andThen 则是将多个偏函数进行**链接**。

