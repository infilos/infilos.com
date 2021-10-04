---
type: docs
title: "Primitive"
linkTitle: "Primitive"
weight: 15
---

## 基本类型

Byte、Short、Int、Lont、Char称为**整形类型**，与 Double、Float 一起构成整个**数字类型**。这些都定义在`scala`包中。

而 String 是一个由 Char 构成的序列。

包`scala`与`java.lang`会被自动引入 Scala 源文件。

## 字面值

所有这些基本类型都可以用**字面值**表示，通过字面值可以在代码中显式的定义常量。

### 整形字面值

整形类型，即 Byte、Short、Int、Lont、Char，有两种表示形式：十进制和十六进制，十六进制以`0x`或`0X`开头。

无论以何种形式初始化整形字面值，Scala 都会以十进制打印该字面值。

```bash
# 以 16 进制初始化整形字面值
scala> val hex = 0x5 			# hex: Int = 5
scala> val hex2 = 0x00FF 		# hex2: Int = 255
scala> val magic = 0xcafebabe 	# magic: Int = -889275714

# 以 10 进制初始化整形字面值
scala> val dec1 = 31 			# dec1: Int = 31
scala> val dec2 = 255 			# dec2: Int = 255
scala> val dec3 = 20 			# dec3: Int = 20
```

如果一个整形字面值以字母`L`或`l`开头，则为类型`Long`：

```bash
scala> val prog = 0XCAFEBABEL 	# prog: Long = 3405691582
scala> val tower = 35L 			# tower: Long = 35
scala> val of = 31l 			# of: Long = 31
```

如果一个整形字面值被赋值给一个类型为`Short`或`Byte`的变量，则这个字面值为被当做`Short`或`Byte`类型，并且该字面值必须在这些类型的有效取值范围内。

### 浮点数字面值

浮点数字面值由 10 进制数字创建，带有可选的小数点，和一个`E`或`e`及对应的指数。

```bash
scala> val big = 1.2345 		# big: Double = 1.2345
scala> val bigger = 1.2345e1 	# bigger: Double = 12.345
scala> val biggerStill = 123E45 # biggerStill: Double = 1.23E47
```

同时，可以用结尾字符`D`或`d`表示双精度浮点，`f`或`F`表示单精度浮点。

### 字符字面值

字符字面量由使用单引号包围的任意 Unicode 字符构成。

```bash
scala> val a = 'A' 				# a: Char = A
```

同时可以使用以`\u`开头的 Unicode 码表示：

```bash
scala> val d = '\u0041' 		# d: Char = A
```

同时可以在任意位置使用 Unicode 字符：

```bash
scala> val B\u0041\u0044 = 1 	# BAD: Int = 1
```

### 字符串字面值

字符串字面值是一个由双引号包围的字符字面值序列。

同时，可以定义多行字符串：

```scala
println("""Welcome to Ultamix 3000. 
			Type "HELP" for help.""")
println("""|Welcome to Ultamix 3000. 
		   |Type "HELP" for help.""".stripMargin)
```

### 符号字面值(Symbol)

符号字面值写作`'indent`，`indent`部分可以任意字符数字标示符。这样的一个标示符被被映射为`scala.Symbol`的一个实例，编译器会将它编译为一个工厂方法调用：`Symbol("indent")`。

字符字面量并不能做太多操作，只能获取它的`name`属性：

```bash
scala> val s = 'aSymbol 		# s: Symbol = 'aSymbol
scala> val nm = s.name 			# nm: String = aSymbol
```

同时需要注意的是字符字面量是`interned`，编写一个字符字面量两次，表达式会引用同一个`Symbol`对象。

## 字符串插值

可以使用`s`直接在字符串字面量中引用变量进行插值：

```scala
val name = "reader" 
println(s"Hello, $name!")
  
s"The answer is ${6 * 7}." 		// res0: String = The answer is 42.
```

使用`raw`创建的字符串不会对字面值进行转义：

```scala
println(raw"No\\\\escape!") 	// prints: No\\\\escape!
```

使用`f`创建格式化字符串：

```scala
scala> f"${math.Pi}%.5f" 		// res1: String = 3.14159
```

## 操作符都是方法

基本类型的操作符实际是普通的方法调用：

```scala
// Scala invokes 1.+(2)
scala> val sum = 1 + 2 			// sum: Int = 3
```

`Int`同时包含一些重载方法来接收不同类型的参数：

```scala
// Scala invokes 1.+(2L)
scala> val longSum = 1 + 2L 	// longSum: Long = 3
```

所有的方法都可以作为操作符。中缀操作符(infix)接收两个运算数，一个在左边一个在右边。前缀操作符(prefix)接收一个操作数，位于操作符的右边。而后缀操作符(postfix)则是操作数位于操作符的左边。

```scala
7 + 2		// infix
-7			// prefix
7 toLong	// postfix
```

在前缀操作符中，会将表达式转换成对应的方法调用：

```scala
-2.0
(2.0).unary_-
```

可以作为前缀操作符的标示符只有`+、-、!、~`。因此只有使用这四种标示符类定义方法，比如`unary_!`，才能以`!param`这样的语法调用。

## 对象相等

`==`可以用于所有的对象相等性比较。该方法定义在`Any`包中，实际的意义为：

```scala
final def == (that: Any): Boolean = if (null eq this) {null eq that} else {this equals that}

x == that
if (x eq null) that eq null else x.equals(that)
```

Java 中的`==`可以用于比较基本类型和引用类型。基本类型时会进行值比较，这与 Scala 一致。

但是在比较引用类型时，Java 进行引用相等性比较，即两个变量是否指向 JVM 堆中的同一个对象，Scala 会使用`equals`进行引用类型的比较，该方法由用户定义。

而 Scala 中的引用相等性比较则使用`eq`方法。而 Java 中的`equal`仅作为引用比较。

### 创建比较方法

在定义`equals`方法时，有四种影响判等行为的**陷阱**：

1. `equals`方法签名错误
2. 改变`equals`放但是没有改变`hashCode`方法
3. 依据可变字段定义`equals`方法
4. 没有为`equals`定义正确的等价关系

#### 1、方法签名错误

现在有一个简单的类：

```scala
class Point(val x: Int, val y: Int) { ... }
```

现在是第一种`equals`方法的实现：

```scala
def equals(other: Point): Boolean = 
  this.x == other.x && this.y == other.y
```

进行测试：

```scala
val p1, p2 = new Point(1, 2)
val q = new Point(2, 3)
p1 equals p2		// Boolean = true
p1 equals q			// Boolean = false
```

看起来一切正常，但是把他们放入集合时：

```scala
val coll = mutable.HashSet(p1)
coll contains p2	// Boolean = false
```

虽然`p1`与`p2`相等，但是`contains`方法却判断失败。

同时，当我们把`p2`赋值给一个`Any`类型的对象时：

```scala
val p2a: Any = p2
p1 equals p2a		// Boolean = false
```

比较结果任然错误。

下面是`Any`中的`equals`定义：

```scala
def equals(other: Any): Boolean
```

在一开始我们定义的`equals`方法中，参数类型设置为`Point`而不是`Any`，同时没有对`Any`中的方法进行重写，即使用`override`关键字标识。因此，这只是一个方法重载。当前，Scala 与 Java 中的重载已经通过参数的静态类型解决，但并非运行时类型。因此，当参数的静态类型为`Point`时会调用接收`Poiont`类型参数的方法，一旦参数的静态类型为`Any`，则会调用`Any`类型的方法。

因此在调用`Set`的`contaions`方法时，它会调用`object`类型的泛型`equals`方法而不是`Point`类型的方法。同时也是`p1 equals p2a`失败的原因。

下面是正确的`equals`定义：

```scala
override def equals(other: Any) = other match { 
	case that: Point => this.x == that.x && this.y == that.y 
	case _ => false 
}
```

> 同时以相同的签名重写`==`方法，因为他被定义为`final`。

#### 2、未重新定义 hashCode 方法

现在重复测试`coll contains p2`是仍然会出现错误，但并不是 100%。因为`Set`会以元素的 hash 值来进行比较，但是`Point`并未定义新的`hashCode`方法，仍然是原始的定义：只是对已分配对象的地址的转换。

在调用`equals`结果为`true`后会分别调用两个对象的`hashCode`方法并对结果进行比较。

同时，`hashCode`只能依赖于字段的值。下面是一个正确的定义：

```scala
class Point(val x: Int, val y: Int) { 
	override def hashCode = (x, y).## 
	override def equals(other: Any) = other match { 
		case that: Point => this.x == that.x && this.y == that.y 
		case _ => false 
	}
}
```

`##`方法是用于计算主要类型、引用类型、null的快捷方式。

#### 3、依据可变字段定义 equals 方法

比如下面的定义，字段被定义为`var`而不再是`val`：

```scala
class Point(var x: Int, var y: Int) {		// var
	override def hashCode = (x, y).## 
	override def equals(other: Any) = other match { 
		case that: Point => this.x == that.x && this.y == that.y 
		case _ => false 
	}
}
```

这是在通过`Set`的`contains`方法进行判断：

```scala
val p = new Point(1, 2)
val coll = collection.mutable.HashSet(p)
coll contains p				// true
// 修改 p 的字段值
p.x += 1
coll contains p				// false
coll.iterator contains p	// true
```

如果改变了`p`的字段值，将会判断失败，但是通过`iterator`方法发现`p`仍然是`Set`的元素。

这是因为，修改字段值后的`p`，其 hash 值也跟着改变，`contaions`方法通过 hash 值比较的结果必然失败。

#### 4、错误的等价关系

`scala.Any`的`equals`方法约定中，指定`equals`方法必须为`non-null`对象实现正确的等价关系。

- 反射性：`non-null`值 x，表达式`x.equals(x)`必须返回`true`
- 对称性：任何`non-null`值 x 和 y，当且仅当`x.equals(y)`返回`true`时，`y.equals(x)`才会返回`true`
- 传递性：任何`non-null`值 x、y、z，如果`x.equals(y)`和`y.equals(z)`都返回`true`，则`x.equals(z)`也会返回`true`
- 一致性：任何`non-null`值 x 和 y，多次调用`x.equals(y)`都会一致的返回`true`或`false`
- 对任何`non-null`值 x，`x.equals(null)`应该返回`false`

上面的`Point`类已经能够很好的工作，但是如果它有一个新的子类，并且新增了一个字段：

```scala
object Color extends Enumeration {
	val Red, Orange, Yellow, Green, Blue, Indigo, Violet = Value 
}

class ColoredPoint(x: Int, y: Int, val color: Color.Value)
	extends Point(x, y) {
	override def equals(other: Any) = other match { 
		case that: ColoredPoint => this.color == that.color && super.equals(that) 
		case _ => false 
	}
}
```

通常会以上面的方式实现。这个子类继承父类，并重写了`equals`方法，该方法类似父类的形式，比较新字段并利用父类的`equals`方法比较原有的字段。

注意当前这个例子中，并不需要重写`hashCode`方法，因为子类中的`equals`实现比父类中的实现更为*严谨*(它与更小范围内的对象相等)，因此`hashCode`的契约依然有效。？？？

这个子类中的实现看起来没有问题，但是当他与父类混合时：

```scala
scala> val p = new Point(1, 2) 		
# p: Point = Point@5428bd62
scala> val cp = new ColoredPoint(1, 2, Color.Red) 
# cp: ColoredPoint = ColoredPoint@5428bd62

scala> p equals cp 		# res9: Boolean = true
scala> cp equals p 		# res10: Boolean = false
```

`p equals cp`会调用`p`的`equals`方法，这个方法只会对对象的坐标进行比较，并返回了`true`。

`cp equals p`会调用`cp`的`equals`方法，因为`p`并不是一个`ColoredPoint`对象，因此返回`false`。

因此，`equals`中定义的相等性并不是对称的。

#### canEqual

在继承类型的比较中，需要引入一个`canEqual`方法。这个想法是，一旦一个类重新定义了`equals`(或同时也冲定义了`hashCode`)，它也必须明确指出，这类对象永远不能等于那些实现了不同判等方法的父类对象。

```scala
def canEqual(other: Any): Boolean
```

这个方法中，如果`other`对象是一个(重)定义了`canEqual`方法的类的实例，返回`true`，否则返回`false`。在`equals`方法中调用这个方法来确保将要比较的两个对象能够进行双向比较。

```scala
class Point(val x: Int, val y: Int) { 
	override def hashCode = (x, y).## 
	override def equals(other: Any) = other match { 
		case that: Point => 
			(that canEqual this) && (this.x == that.x) && (this.y == that.y) 
		case _ => false
	} 
	def canEqual(other: Any) = other.isInstanceOf[Point]	// 运行时类型相同
}
```

然后是子类的定义：

```scala
class ColoredPoint(x: Int, y: Int, val color: Color.Value) extends Point(x, y) {
	override def hashCode = (super.hashCode, color).## 		// 重写 hashCode
	override def equals(other: Any) = other match { 
		case that: ColoredPoint => 							// 重写 equals
		(that canEqual this) && super.equals(that) && this.color == that.color 
		case _ => false
	} 
	override def canEqual(other: Any) = other.isInstanceOf[ColoredPoint]
}
```

> 对象相等性的实现依赖于场景。当前场景中，两个不同的`Point`对象拥有相同的坐标即视作相等。但是两个对象拥有相同坐标，但是一个没有颜色，一个为红色，则视作不相等。

[拓展：Java 中的字符串相等性比较](http://www.techug.com/java-language-defect-2-equals-compare-strings)

