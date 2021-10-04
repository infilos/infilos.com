---
type: docs
title: "函数式对象"
linkTitle: "函数式对象"
weight: 24
---

> 函数式对象，即不拥有任何可变状态的对象的类。

## 构建一个分数类

比如我们要构建一个**分数**类，并最终能够实现下面的操作：

```bash
scala> val oneHalf = new Rational(1, 2) 
# oneHalf: Rational = 1/2 scala> 
val twoThirds = new Rational(2, 3) 
# twoThirds: Rational = 2/3 
scala> (oneHalf / 7) + (1 - twoThirds) 
# res0: Rational = 17/42
```

## 设计分数类

设计一个分数类时需要考虑客户端会如何使用该类来创建实例。同时，我们把分数类的实例设计成不可变对象，并在创建实例时提供所有需要的参数，这里指 分数和分母。

```scala
class Rational(n:Int, d:Int)
```

## 重新实现 toString 

使用上面的类创建实例时：

```bash
scala> new Rational(1, 2)
res0: Rational = Rational@2591e0c9
```

默认情况下，一个类会继承`java.String.Object`中的`toString`实现。然而，为了更好的使用`toString`方法，比如日志、错误追踪等，我们需要实现一个更加详细的方法，比如包含该类的字段值。通过`override`来重写：

````scala
class Rational(n:Int, d:Int) {
  override def toString = n + "/" + d
}
````

现在就可以获得更详细的信息了：

```bash
scala> val x = new Rational(1, 3) 
x: Rational = 1/3
```

> 这里需要注意的是，Java 中的类拥有构造器，并使用构造器来接收构造参数。在 Scala 中，类可以直接接收参数，称为类参数，这些类参数能够在类体中直接使用。
>
> 如果类参数使用 val 或 var 声明，它们同时成为类的可变或不可变字段，但是如果不适用任何 var 或 val，这些类参数不会成为类的成员，只能在类内部引用。也即本例中的使用方式。

## 检查前提条件

事实上，分数中分母是不能为 0 的，但是我们的主构造器中没有任何处理。如果使用了 0 作为分母，后续的处理中将会出现错误。

```bash
scala> new Rational(5, 0) 
res1: Rational = 5/0
```

面向对象语言的一个优势就是可以讲数据封装到一个对象，因此可以在该对象整个生命周期中确保数据的状态。在一个不可变对象中，比如这里的`Rational`，要确保它的状态，就要求在一开始构造的时间对数据做充分的验证，因为一旦创建就不会再进行改变。因此我们可以通过`require`在其主构造器中定义一个**前提条件**：

```scala
class Rational(n:Int, d:Int){
  require(d != 0)
  override def toString = n + "/" + d
}
```

这时，如果在构造时传入一个 0 作为分母，`require`则会抛出一个`IllegalArgumentException`异常。

## 加法操作

现在我们实现`Rational`的加法操作，实际也就是其字段的加法操作。因为它是一个不可变类，因此不能在一个`Rational`对象本身进行操作，而应该创建一个新的对象。

或许我们可以这样实现：

```scala
class Rational(n: Int, d: Int) { // This won't compile 
	require(d != 0) 
	override def toString = n + "/" + d 
	def add(that: Rational): Rational = 
		new Rational(n * that.d + that.n * d, d * that.d) 
}
```

但是当我们尝试编译时：

```bash
<console>:11: error: value d is not a member of Rational 
			new Rational(n * that.d + that.n * d, d * that.d)
								  ^
```

尽管类参数 n 和 d 在`add`方法的作用域中，但是`add`方法只能访问调用对象自身的值。因此，`add`方法中，可以访问并使用 n 和 d 的值。但是却不能使用`that.n`和`that.d`，因为`that`并不是`add`方法的调用者，只是作为`add`方法的参数。如果想要使用`that`的类参数，需要将这些参数放在字段中，以支持使用实例来引用：

```scala
class Rational(n:Int, d:Int){
  	require(d != 0) 
  	val numer:Int = n
  	val denom:Int = d
	override def toString = numer + "/" + denom
	def add(that:Rational): Rational = 
		new Rational(
			numer * that.denom + that.numer * that.denom,
			denom * that.denom
		)
}
```

同时需要注意的时，之前使用类参数的方式来构造对象，但是并不能在外部访问这些类参数，现在可以直接访问类的字段：

```bash
scala> r.numer 	# res3: Int = 1
scala> r.denom 	# res4: Int = 2
```

## 自引用

关键字`this`指向当前执行方法被调用的对象实例，或者如果使用在构造器内时，指正在被构建的对象实例。

比如添加一个`lessThan`方法，测试当前分数是否小于传入的参数：

```scala
def lessThan(that:Rational) = 
	this.numer * that.denom < that.numer * this.denom
```

这里的`this`指调用`lessThan`方法的实例对象，也可以省略不写。

再比如添加一个`max`方法，比较当前对象与传入参数那个更大，并返回大的那一个：

```scala
def max(that:Rational) = 
	if (this.lessThan(that)) that else this
```

这里的`this`就不能省略了。

## 辅助构造器

Scala 中朱构造器之外的构造器称为辅助构造器。比如创建一个分母为 1 的分数，可以实现为只需要提供一个分子，分母默认为 1:

```scala
class Rational(n:Int, d:Int){
  require(d != 0)
  
  val numer:Int = n
  val denom:Int = d
  
  def this(n:Int) = this(n, 1)	// 辅助构造器
  ....
}
```

辅助构造器的函数体这是对朱构造器的调用。Scala 中的每个辅助构造器都是调用当前类的其他构造器，可以是主构造器，也可以使已定义的其他辅助构造器。因此最终也都是对主构造器的调用，**主构造器是类的唯一入口点**。

> Java 中构造器能够调用同类的其他构造器或超类构造器。Scala 中只有主构造器可以调用超类构造器。

## 私有字段和方法

分数 66/42 并不是最简化形式，简化过程就是求最大公约数的过程，比如我们定义一个私有字段 g 表示当前分数的最大公约数，定义一个私有方法 gcd 来求最大公约数：

```scala
class Rational(n:Int, d:Int){
  ...
  private val g = gcd(n.abs, d.abs)
  val numer = n /g
  val denum = d /g
  private def gcd(a:Int, b:Int):Int = if (b ==0) a else gcd(b, a % b) // 辗转相除
  ...
}
```

## 定义操作符

使用 + 来作为求和的方法名，而不是 add。同时定义乘法操作：

```scala
class Rational(n:Int, d:Int){
  ...
  def +(that:Rational): Rational = 
  	new Rational(
  	  number * that.denom + that.numer* denom,
  	  denom * that.denom
  	)
  	
  def *(that:Rational): Rational = 
  	new Rational(numer * that.numer, denom * that.denom)
  
  ...
}
```

以操作符来组合调用时仍然会按照运算操作符的优先级进行。

## 标识符

**字母数字下划线标识符**，以字母数字或下划线开始，后跟字母数字下划线。`$`同样被当做字符，但是被保留作为编译器生成的标识符，因此不做他用。

遵循**驼峰命名**，避免使用下划线，特别是结尾使用下划线。

常量使用大写并用下划线分割单词。

## 方法重载

比如分数和整数不能直接做除法，需要首先将整数转换为分数，`r * new Rational(2)`，这样很不美观，因此可以创建新的方法来直接接受整数来进行乘法运算：

```scala
def * (that: Rational): Rational = 
	new Rational(numer * that.numer, denom * that.denom)

def * (i: Int): Rational = new Rational(numer * i, denom)
```

## 隐式转换

但是如果先要以`2 * r`的方式进行运算，但是整数并没有一个接受`Rational`实例作为参数的方法，因此我们可以定义一个隐式转换，将整数在需要的时候自动转换为一个分数实例：

```scala
implicit def intToRational(x: Int) = new Rational(x)
```



