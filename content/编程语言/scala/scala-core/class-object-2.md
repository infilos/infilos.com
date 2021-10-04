---
type: docs
title: "Class Object: 类"
linkTitle: "Class Object: 类"
weight: 21
---

## 创建主构造器

Scala 的主构造器包括：

- 主构造器参数
- 类体中调用的方法
- 类体中执行的语句和表达式

```scala
class Person(var firstName:String, var lastName:String){
  println("the constructor begins")
  
  // 字段
  private val HOME = System.getProperty("user.home")
  var age = 0
  
  // 方法
  override def toString = s"$firstName $lastName is $age years old"
  def printHome = { println(s"HOME = $HOME") }
  def printFullName = { println(this) } // uses toString
  
  // 方法调用
  printHome
  printFullName
  println("still in the constructor")
}
```

这个例子中，整个类体中的部分都属于构造器，包括字段、表达式执行、方法、方法调用。

在 Java 中你可以清晰的区分是否在主构造器之中，但是 Scala 模糊了这种区分。

- 构造器参数：这里被声明为 var，表示两个字段是可变字段，类体中的 age 字段也是一样，而 HOME 字段没声明为 val，表示为不可变字段
- 方法调用：类体中的方法调用同样属于主构造器的一部分

## 控制构造器字段可见性

影响字段可见性的几种因素：

- 如果字段声明为 var，会同时生成 setter 和 getter 方法
- 如果字段声明为 val，只会生成 getter 方法
- 如果既没有 val 也没有 var，Scala 会以保守的方式不生产任何 setter 或 getter
- 如果字段声明为 private，无论是 var 还是 val，都不会生成任何 setter 或 getter

而 case 类默认指定字段为 val。

## 定义辅助构造器

可以同时定义多个辅助构造器，以支持使用不同的方式创建类实例。定义的方式为，使用 this 方法创建辅助构造器，并且所有的辅助构造器需要拥有不同的签名(参数列表)，并且，每个辅助构造器都必须调用以定义的上个构造器。

```scala
// 主构造器
class Pizza(var crustSize:Int, var crustType: String){
  // 一个参数的辅助构造器
  def this(crustSize:Int){
    this(crustSize, Pizza.DEFAULT_CRUST_TYPE)
  }
  
  // 一个参数的辅助构造器
  def this(crustType:String) {
    this(Pizza.DEFAULT_CRUST_SIZE, crustType)
  }
  
  // 无参数辅助构造器
  def this() {
    this(Pizza.DEFAULT_CRUST_SIZE, Pizza.DEFAULT_CRUST_TYPE)
  }
  
  override def toString = s"A $crustSize inch pizza with a $crustType crust"
}

object Pizze {
  val DEFAULT_CRUST_SIZE = 12
  val DEFAULT_CRUST_TYPE = "THIN"
}
```

然后就可以使用不同的方式来创建实例：

```scala
val p1 = new Pizza(Pizza.DEFAULT_CRUST_SIZE, Pizza.DEFAULT_CRUST_TYPE) 
val p2 = new Pizza(Pizza.DEFAULT_CRUST_SIZE) 
val p3 = new Pizza(Pizza.DEFAULT_CRUST_TYPE) 
val p4 = new Pizza
```

**辅助构造器的使用要点：**

- 使用 this 方法定义
- 都必须调用在前面已定义的辅助构造器
- 都必须拥有不同的签名
- 每个构造器使用 this 调用其他的构造器

### 为 case 类创建辅助构造器

case 类会比一般的类为你生成更多的模板代码，并且，case 类的构造器并不是真正的构造器，而是伴生对象中的 apply 方法，因此辅助构造器的定义方式也有所不同。

```scala
case class Pserson(var name:String, var age:Int)

// 伴生对象
object Person{
  def apply() = new Person("<no name>", 0)
  def apply(name:String) = new Person(name, 0)
}

Person()
Person("Pam")
Persion("Alex", 10)
```

## 定义私有构造器

有时候需要定义一个私有构造器，比如在实现单例模式的时候：

```scala
class Order private { ... }
class Person private (name:String) { ... }
```

这时只能在类的内部或半生对象中创建实例：

```scala
object Person{
  val person = new Person("Alex")
  def getInstance = person
}

some where {
  val person = Person.getInstance
}
```

这样就实现了单例模式。大多数时候并不需要使用私有构造器，通常只需要定义一个 object。

## 为构造器参数提供默认值

```scala
class Socket(val timeout:Int = 10000)
```

这种方式实际上是有两个构造器组成：一个单参数的主构造器，一个无参数的辅助构造器。

```scala
class Socket(val timeout: Int) {
  def this() = this(10000) 
  override def toString = s"timeout: $timeout"
}
```

## 重写默认的访问器和修改器

比如一个 Person 类：

```scala
class Person(private var name: String) { 
  // 实际上是创建了一个循环引用
  def name = name 
  def name_=(aName: String) { name = aName } 
}
```

这会导致编译错误，因为 Scala 已经自动生成了**同名**的 getter 和 setter 方法，如果再创建同名的方法，实际上是一个循环引用，并导致编译失败。遍历的方式是修改主构造器中的参数名，而在自定义的 setter 和 getter 方法中使用真正有用的参数名：

```scala
class Person(private var _name: String) { 
  def name = _name								// getter
  def name_=(aName: String) { _name = aName }	// setter
}
```

注意，参数必须被声明为 private，因此只能通过 setter 和 getter 方法来设置和访问字段值。

## 避免自动生成 setter 和 getter

Scala 会自动为主构造器参数生成 setter 和 getter 方法，如果不想生成这些方法：

```scala
class Stock {
// getter and setter methods are generated 
  var delayedPrice: Double = _

// keep this field hidden from other classes 
  private var currentPrice: Double = _
}
```

### 私有字段

如果一个字段被声明为 private，则只有类的实例能够访问该字段，或者类的实例访问该类的其他实例的这个字段：

```scala
class Stock {
  private var price: Double = _ 
  def setPrice(p: Double) { price = p } 
  def isHigher(that: Stock): Boolean = this.price > that.price 
}

object Driver extends App { 
  val s1 = new Stock 
  s1.setPrice(20) 
  val s2 = new Stock 
  s2.setPrice(100) 
  println(s2.isHigher(s1)) // s2 的 isHigher 方法访问了 s1 的私有字段 price
}
```

### 对象私有字段

如果使用`private[this]`来修饰字段会进一步增强该字段的私密性，被修饰的字段只能被当前对象访问，与普通的 private 不同，这种方式使相同类的不同实例也不能访问该字段。

## 使用代码块或函数为字段赋值

类字段可以通过一段代码块或一个函数来赋值。这些操作都属于构造器的一部分，只有在该类创建新的实例时执行。

```scala
class Foo{
  val text = { var lines = "" try {
	lines = io.Source.fromFile("/etc/passwd").getLines.mkString } catch {
	  case e: Exception => lines = "Error happened" 
	} 
	lines
  }
}
```

如果使用了 lazy 关键字来修饰字段，则只有该字段在第一次被访问时才会进行初始化。

## 设置未初始化的可变字段类型

通常的方式是使用 Option 并设置为 None，对于一些基本类型，可以设置为常用的默认值。

```scala
case class Person(var name:String, var password:String){
  var age = 0
  var firstName = ""
  var lastName = ""
  var address = None: Option[Address]
}
```

但是，推荐的方式是设置字段为 val 类型，并在需要的时候使用 copy 方法根据原有的对象创建一个新对象而不是修改原有对象的字段值，同时，避免使用 null 和初始值，应尽量使用 Option 并设置 None 来作为默认值。

## 类继承时如何处理构造器参数

在子类继承基类时，由于 Scala 已经为基类的构造器参数自动生成了 setter(var) 和 getter 方法，因此子类在声明构造器参数时，可以省略掉参数前面的 var 或 val，以避免重新自动生成的 setter 和 getter 方法。

```scala
class Person(var name:String, var age:Int){
  ...
}

class Employe(name:String, age:Int, var gender:Int) extends Person(name, age){
  ...
}
```

## 调用父类构造器

可以在子类的主构造器中调用父类构造器或不同的辅助构造器，但是不能在子类辅助构造器中调用父类构造器。

```scala
class Animal(var name:String, var age:Int){
  def this(name:String){
    this(name, 0)
  }
}

class Dog(name:String, age:Int) extends Animal(name,age){
  ...
}

class Dog(name:String) extends Animal(name, 0){
  ...
}
```

因为任何类的辅助构造器都要调用类自身中已定义的其他构造器，因此也就不能调用父类的构造器了。

## 何时使用抽象类

由于 Scala 中拥有特质，比抽象类更加轻量且支持线性扩展(允许混入多个特质，但是不能继承多个抽象类)，只有很少的需求来使用抽象类：

- 对构造器参数有需求的时候，因为特质没有构造器
- 这部分代码会被 Java 调用

抽象类语法：

```scala
abstract class BaseController(db:Database) {
  def save { db.save }
  def update { db.update }
  
  def connect
  def getStatus:String
  def setServerName(serverName:String)
}
```

继承自抽象类的类要么实现所有的抽象方法，要么也声明为抽象类。

## 在抽象基类或特质中定义属性

可以在抽象类或特质中使用 var 或 val 来定义属性，以便在所有子类中都能访问：

- 一个抽象的 var 字段会自动生成 setter 和 getter 字段
- 一个抽象的 val 字段会自动生成 getter 字段
- 定义抽象字段时，并不会在编译后的结果代码中创建这些字段，只是自动生成对应的方法，因此在子类中仍然要使用 val 或 var 来定义这些字段，但是如果抽象类中已经提供了字段的默认值，子类中就不需要再使用 var 或 val 来修饰字段，可以根据需要直接修改字段值

同时，抽象类中不应该使用 null，而应该使用 Option。

## 通过 case 类生成模板代码

使用 case 类会自动创建一系列模板代码：

- 生成一个 apply 方法，因此不需要使用 new 关键字创建实例
- 生成 getter 方法，因为 case 类的参数默认为 val，如果生命为 var，则会自动生成 setter 方法
- 生成一个好用的 toString 方法
- 生成一个 unapply 方法，以便能够很好的用于模式匹配
- 生成 equals 和 hashCode 方法
- 生成 copy 方法


## 定义一个 equal 方法(对象相等性)

定义一个 equal 方法用于比较实例之间的相等性：

````scala
class Person(name:String, age:Int){
  def canEqual(a:Any) = a.isInstanceOf[Person]
  
  override def equals(that:Any): Boolean = {
    that match{
      case that:Person => that.canEqual(this) && this.hashCode == that.hashCode
      case _ => false
    }
  }
  
  override def hashCode:Int = {
    val prime = 34
    var result = 1
    result = prime * result + age; 
    result = prime * result + (if (name == null) 0 else name.hashCode) 
    return result
  }
}
````

因为定义了 canEqual 方法，因此可以使用`==`来比较实例之间的相等性，与 Java 不同的是，`==`在 Java 中是引用的比较。

### 原理

Scala 文档中对任何类中 equal 方法的要求：任何该方法的实现必须是**值相等性关系**。必须包含以下三个属性：

- 它是反射的：任何类型的实例 x，`x.equals(x)`必须返回 true
- 它是对称的：任何类型的实例 x 和 y，当且仅当`y.equals(x)`返回 true 时，`x.equals(y)`返回 true
- 它是传递的：任何类型的实例 x、y、z，如果`x.equals(y)`和`y.equals(z)`都返回 true，`x.equals(z)`也必须返回 true

## 创建内部类

```scala
class PandorasBox { 
  case class Thing (name: String) 
  var things = new collection.mutable.ArrayBuffer[Thing]() 
  things += Thing("Evil Thing #1") 
  things += Thing("Evil Thing #2")

  def addThing(name: String) { things += new Thing(name) }
}
```

外部对 Thing 一无所知，只能通过 addThing 方法来添加。在 Scala 中，内部类会绑定到外部对象上，而不是一个单独的类。