---
type: docs
title: "组合继承"
linkTitle: "组合继承"
weight: 26
---

类之间的两种关系：组合、继承。组合即持有另一个类的引用，借助被引用的类完成任务。继承是超类与子类的关系。

## 实例：二维布局库

目标是建立一个创建和渲染二维元素的库。每个元素都将显示一个由文字填充的矩形，称为 Element。提供一个工厂方法 elem 来通过传入的数据构建新元素：

```scala
elem(s:String):Element
```

可以对元素调用 above 和 beside 方法并传入第二个元素，来获取一个将二者合并后生成的新元素：

```scala
val column1 = elem("hello") above elem("***")
val cloumn2 = elem("***") above elem("workd")
```

获得的结果为：

```scala
hello *** 
 *** world
```

above 和 beside 可以称为组合操作符，或连接符，它们把某些区域的元素组合成新的元素。

## 抽象类

Element 代表布局元素类型，因为元素是二维的字符矩形，因此它包含一个 content 成员表示元素内容。内容有字符串数组表示，每个字符串代表一行：

```scala
abstract class Element{
  def contents: Array[String]
}
```

## 定义无参数方法

需要向 Element 添加显示高度和宽度的方法，height 返回 contents 的行数，也就表示高度，width 返回第一行的长度，没有元素则返回 0。

```scala
abstract class Element{
  def contents: Array[String]
  def height:Int = contents.length
  def width:Int = if (height == 0) 0 else contents(0).length
}
```

这三个方法都是无参方法，甚至没有空的参数列表括号。

> 如果方法中不需要参数，并且，方法只能通过读取所包含的对象的属性去访问可变状态(即方法本身不能改变可变状态)，就使用无参方法。
>
> 这一惯例支持**统一访问原则**，即客户端不应由属性是通过方法实现还是通过字段实现而受影响(访问字段与调用无参方法看上去没有差别)。
>
> 如果是直接或间接的使用了可变对象，应该使用空的括号，以此来说明该调用触发了计算。

比如，可以直接将 height 方法和 width 方法改成字段实现的形式：

```scala
abstract class Element{
  def contents: Array[String]
  val height:Int = contents.length
  val width:Int = if (height == 0) 0 else contents(0).length
}
```

客户端不会感觉到任何差别。唯一的区别是访问字段比调用方法略快，因为字段值在类初始化的时候被预计算，而方法调用在每次调用的时候都要计算。同时，使用字段需要为每个 Element 对象分配更多的存储空间。

如果需要将字段改写为方法时，方法是由**纯函数**构成，即没有副作用也没有可变状态，那么客户端代码就不需要做出改变。

## 扩展类

为了穿件 Element 对象，我们需要实现一个子类扩展抽象类 Element 并实现其抽象方法 contents。

```scala
class ArrayElement(conts:Array[String]) extends Element{
  def contents: Array[String] = conts
}
```

关键字 extends 的作用：使 ArrayElement 类继承了 Element 的所有非私有成员，并成为其子类。

## 重写方法和字段

Scala 中的字段和方法属于相同的命名空间。字段可以重写无参数方法。比如父类中的 contents 是一个无参方法，可以在子类中重写为一个字段而不需要修改父类中的定义：

```scala
class ArrayElement(conts:Array[String]) extends Element{
  val contents: Array[String] = conts
}
```

同时，禁止在一个类中使用相同的名称定义方法和字段。这在 Java 中是支持的，因为 Java 提供了四个命名空间：字段、方法、类型、包。但是 Scala 中仅提供两个命名空间：

- 值（字段、方法）
- 类型（类、特质名）

## 定义参数化字段

上面 ArrayElement 类的构造器参数 conts 的实际作用是将值复制给 contents 字段，这里存在了冗余，因为 conts 实际上就是 contents，只是取了一个与 contents 类似的变量名以作区分，实际上可以使用参数化字段，而不需要再进行多余的传递：

```scala
class ArrayElement(val contents: Array[String]) extends Element
```

构造器中的 val，是同时定义同名的参数和字段的简写方式，同时，这个 contents 被定义为一个不可变字段，并且使用参数初始化。如果使用 var 来定义，则该字段是一个可变字段。

对于这样参数化的字段，同样可以进行重写，同时也能使用可见性修饰符：

```scala
class Cat {
  val dangerous = false
}

class Tiger(
  override val dangerous:Boolean,
  private var age:Int
) extends Cat
```

这个例子中，子类 Tiger 通过参数化字段的方式重写了父类中的字段 dangerous，同时定义了一个私有字段 age。或者以更完整的方式：

```scala
class Tiger(param1:Boolean, param2:Int) extends Cat(
  override val dangerous:Boolean = pararm1,
  private var age = param2
)
```

这两个 Tiger 的实现是等效的。

## 调用超类构造器

现在系统中已经有了两个类：Element 和 ArrayElement。如果客户想要创造由单行字符串构成的布局元素，我们可以实现一个子类：

```scala
class LineElement(s:String) extends ArrayElement(Array(s)) {
  override def width = s.length
  override def height = 1
}
```

因为子类 LineElement 要继承 ArrayElement，但是 ArrayElement 有一个参数，这时 LineElement 需要给超类的构造器传递一个参数。

**需要调用超类的构造器，只需要把要传递的参数列表放在超类之后的括号里即可。**

## 使用 override 修饰符

如果子类成员重写父类具体成员，则必须使用 override 修饰符；如果父类中是抽象成员时，可以省略；如果子类未重写或实现基类中的成员，则禁用该修饰符。

常用习惯是，重写或实现父类成员时均使用该修饰符。

## 多态和动态绑定

前面的例子中：

```scala
val elem:Element = new ArrayElement(Array("hello","world"))
```

这样将一个子类的实例赋值给一个父类的变量应用，称为**多态**。这种情况下，Element 可以有多种形式，现在已经定义的有 ArrayElement 和 LineElement，可以通过继承 Element 来实现更多的形式。比如，下面定义一个拥有给定长度和高度并通过提供的字符进行填充的实现：

```scala
class UniformElement(
  ch:Char,
  override val width:Int,
  override val height:Int
) extends Element{
  private val line = ch.toString *width
  def contents = Array.make(height, line)
}
```

现在，Element 类型的变量可以接受多种子类的实现：

```scala
val e1:Element = new ArrayElement(Array("hello","world"))
val ae:ArrayElement = new LineElement("hello")
val e2:Element = ae
val e3:Element = new UniformmElement('x',2,3)
```

另一方面，变量和表达式上的方法调用是动态绑定的。被调用的实际方法取决于运行期对象基于的类，而不是变量或表达式的类型。

## 定义 final 成员

有时需要确保一个成员不会被子类重写，这时可以使用 final 修饰符限定。

或者有时候需要确保整个类都不会有子类，也可以在类的声明上添加 final 修饰符。

## 使用组合与继承

组合与继承是使用其他现存类定义新类的两种方法。如果追求的是根本上的代码重用，通常推荐采用组合而不是继承。组合可以避免脆基类的问题，因为在修改基类时会在无意中破换子类。

在使用继承时需要确定，是否建模了一种 “is-a” 的关系，同时，客户端是否想把子类型当做超类来用。

## 实现 above、beside、toString

在 Element 中实现 above 方法，将一个元素放在另一个上面：

```scala
def above(that:Element):Element = {
  new ArrayElement(this.contents ++ that.contents)
}
```

实现 beside 方法，把两个元素靠在一起生成一个新元素，新元素的每一行都来自原始元素的相应行的串联(这里先假设两个元素的长度相同)：

```scala
def beside(that:Element):Element = {
  val contents = new Array[String](this.contents.length)
  for(i <- 0 until this.contents.length)
  	contents(i) = this.contents(i) + that.contents.(i)
  new ArrayElement(contents)
}
```

或者以更简洁的方式实现：

```scala
def beside(that:Element):Element = new ArrayElement(
  for(
    (line1, line2) <- this.contents zip that.contents
  ) yiied line1 + line2
)
```

然后实现一个 toString 方法：

```scala
override def toString = contents.mkString("\n")
```

最后的 Element 实现：

```scala
abstract class Element{
  def contents:Array[String]
  def width:Int = if(height == 0) 0 else contents(0).length
  def height:Int = contents.length
  def above(that:Element):Element = {
    new ArrayElement(this.contents ++ that.contents)
  }
  def beside(that:Element):Element = new ArrayElement(
    for(
      (line1, line2) <- this.contents zip that.contents
    ) yiied line1 + line2
  )
  override def toString = contents.mkString("\n")
}
```

## 定义工厂对象

现在已经拥有了布局元素的类层级，可以将这些层级直接暴露给用户使用，或者可以把这些层级隐藏在工厂对象之后，在工厂对象中包含构建其他对象等方法，客户使用这些工厂方法构建对象而不是直接使用 new 关键字和各层级类来构建对象。

比如在伴生对象中提供工厂方法：

```scala
object Element{
  def elem(contents:Array[String]):Element = new ArrayElement(contents)
  def elem(chr:Char,width:Int,height:Int):Element = new UniformElement(chr,wirdh,height)
  def elem(line:String):Element = new LineElement(line)
}
```

为了能够直接使用 elem 方法而不是 Element.elem，可以直接在 Element 定义文件的头部显示引入该方法，然后对 Element 的实现进行简化：

```scala
import Element.elem

abstract class Element{
  def contents:Array[String]
  def width:Int = if(height == 0) 0 else contents(0).length
  def height:Int = contents.length
  def above(that:Element):Element = {
    elem(this.contents ++ that.contents)
  }
  def beside(that:Element):Element = elem(
    for(
      (line1, line2) <- this.contents zip that.contents
    ) yiied line1 + line2
  )
  override def toString = contents.mkString("\n")
}
```

既然有了工厂方法，所有的子类都可以为私有类，引文他们不再需要直接被客户端使用。可以在类和单例对象的内部定义其他的类和单例对象，因此，可以将 Element 的子类放在其单例对象中实现这些子类的私有化：

```scala
object Element{
  private class ArrayElement(val contents:Array[String]) extends Element
  private class LineElement(s:String) extends Element{
    val contents = Array(s)
    override def width = s.length
    override def height = 1
  }
  private class UniformElement(
    ch:Char,
    override val width:Int,
    override val height:Int,
  ) extends Element{
    private val line = ch.toString * width
    def contents = Array.make(height, line)
  }
  def elem(contents:Array[String]):Element = new ArrayElement(contents)
  def elem(chr:Char,width:Int,height:Int):Element = new UniformElement(chr,width,height)
  def elem(line:String):Element = new LineElement(line)
}
```

