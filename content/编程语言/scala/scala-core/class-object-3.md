---
type: docs
title: "Class Object: 方法"
linkTitle: "Class Object: 方法"
weight: 22
---

## 控制方法作用域

Scala 方法默认为 public，可见性的控制方法与 Java 类似，但是提供比 Java 更细粒度更有力的控制方式：

- 对象私有(object-private)
- 私有(private)
- 包(package)
- 指定包(package-specific)
- 公共(public)

### 对象私有域

只有该对象的当前实例能够访问该方法，相同类的其他实例无法访问：

```scala
private[this] def isFoo = true
```

### 私有域

该类或该类的所有实例都能访问该方法：

```scala
private def isFoo = true
```

### 保护域

只有子类能够访问该方法：

```scala
protected def breathe {}
```

在 Java 中，该域方法可以被当前包(package)的其他类访问，但是在 Scala 中不可以。

### 包域

当前包中所有成员都可以访问：

```scala
package com.acme.coolapp.model {
  class Foo { 
    private[model] def doX {} 	// 定义为包域，model 包中所有成员可以访问
    private def doY {}
  }
}
```

### 更多级别的包控制

```scala
package com.acme.coolapp.model { 
  class Foo { 
    private[model] def doX {}		// 指定到 model 级别
    private[coolapp] def doY {} 	// 指定到 coolapp 级别
    private[acme] def doZ {}		// 指定到 acme 级别
  }
}
```

### 公共域

如果没有任何作用域的声明，即为公共域。

## 调用父类方法

可以在子类中调用父类或特质中已存在的方法来复用代码：

```scala
class WelcomeActivity extends Activity { 
  override def onCreate(bundle: Bundle) { 
    super.onCreate(bundle) 
    // more code here ... 
  } 
}
```

### 指定调用不同特质中的方法

如果同时继承了多个特质，并且这些特质都实现了相同的方法，这时不但能指定调用方法名，还能指定调用的特质名：

```scala
trait Human {
  def hello = "the Human trait" 
}

trait Mother extends Human {
  override def hello = "Mother" 
}

trait Father extends Human {
  override def hello = "Father" 
}

class Child extends Human with Mother with Father { 
  def printSuper = super.hello 
  def printMother = super[Mother].hello 
  def printFather = super[Father].hello 
  def printHuman = super[Human].hello 
}
```

但是并不能跨级别的调用，比如：

```scala
trait Animal
class Pets extends Animal
class Dog extends Pets
```

这时 Dog 只能指定 Pets 中的方法，不能再指定 Animal 中的方法，除非显示继承了 Animal。

## 指定默认参数值

```scala
def makeConnection(timeout: Int = 5000, protocol: = "http") { 
  println("timeout = %d, protocol = %s".format(timeout, protocol)) 
  // more code here 
}
c.makeConnection() 			// 括号不能省略，除非方法定义中没有参数
c.makeConnection(2000) 
c.makeConnection(3000, "https")
```

如果方法有一个参数为默认，而其他参数并没有提供默认值：

```scala
def makeConnection(timeout: Int = 5000, protocol: String)
// error: not enough arguments for method makeConnection:
c.makeConnection("https") 
```

这时任何只提供一个参数值的调用都会报错，可以将定义中带有默认值的参数放在后面，然后就可以通过一个参数来调用：

```scala
def makeConnection(protocol: String, timeout: Int = 5000)
makeConnection("https")
```

## 调用时提供参数名

```scala
methodName(param1=value1, param2=value2, ...)
```

通过参数名提供参数时，参数顺序没有影响。

## 方法返回值为元组

```scala
def getStockInfo = { 
  // other code here ... 
  ("NFLX", 100.00, 10) // this is a Tuple3 
}

val (symbol, currentPrice, bidPrice) = getStockInfo
val (symbol:String, currentPrice:Double, bidPrice:Int) = getStockInfo
```

## 无括号的访问器方法调用

```scala
class Pizza { 
  // no parentheses after crustSize 
  def crustSize = 12 
}
val p = new Pizza
p.crustSize
```

推荐的策略是在调用没有副作用的方法时使用无括号的方式调用。

在纯的函数式编程中不存在副作用，副作用包括：

- 写入或打印输出
- 读取输入
- 修改作为输入的变量的状态
- 抛出异常，或错误发生时终止程序
- 调用其他有副作用的函数

## 接收多变量参数

```scala
def printAll(strings: String*) {
  strings.foreach(println) 
}
printAll("a","b","c")
val list = List(1,2,3)
printAll(list:_*)
```

如果方法拥有多个参数，其中一个是多变量，则这个参数要放在参数列表的末端：

```scala
def printAll(i: Int, strings: String*)
```

## 声明一个能够抛出异常的方法

如果想要声明一个方法，该方法可能会抛出异常：

```scala
@throws(classOf[Exception]) 
override def play {
  // exception throwing code here ... 
}

@throws(classOf[IOException]) 
@throws(classOf[LineUnavailableException]) @throws(classOf[UnsupportedAudioFileException])
def playSoundFileWithJavaAudio {
  // exception throwing code here ... 
}
```

作用是用于提醒调用者或者与 Java 集成。

## 支持流式风格编程

如果想要支持调用者以流式方式调用，即方法链接，如下面的方式：

```scala
person.setFirstName("Leonard").setLastName("Nimoy")
		.setAge(82) 
		.setCity("Los Angeles") 
		.setState("California")
```

为了支持这种方式，需要遵循以下原则：

- 如果你的类会被继承，指定`this.type`作为方法返回值类型
- 如果确定你的类不会被继承，你可以直接在方法中返回`this`

```scala
class Person { 
  protected var fname = "" 
  protected var lname = "" 
  def setFirstName(firstName: String): this.type = { 
    fname = firstName 
    this 
  }
  def setLastName(lastName: String): this.type = {
    lname = lastName 
    this 
  }
}

class Employee extends Person { 
  protected var role = "" 
  def setRole(role: String): this.type = { 
    this.role = role
    this 
  } 
  override def toString = {
	"%s, %s, %s".format(fname, lname, role) 
  }
}
```

然后我们就可以以流式的风格调用方法：

```scala
object Main extends App { 
  val employee = new Employee 
  // use the fluent methods 
  employee.setFirstName("Al") 
  			.setLastName("Alexander") 
  			.setRole("Developer") 
  println(employee)
}
```

如上面的原则所述，如果确定这个类不会被继承，并不需要在 set* 类型的方法中指定`this.type`作为返回值类型，这种情况可以省略，只需要在方法中返回 this 的引用即可。