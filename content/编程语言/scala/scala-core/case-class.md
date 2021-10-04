---
type: docs
title: "Case Class"
linkTitle: "Case Class"
weight: 1
---

下面会用到的 case 类实例：

```scala
case class Person(lastname: String, firstname: String, birthYear: Int)
```

## 基本特性

case 类与普通类的最大区别在于，编译器会为 case 类添加更多的额外特性。

- 创建一个类和它的伴生对象

- 创建一个名为`apply`的工厂方法。因此可以在创建实例时省略掉`new`关键字：

  ```scala
  val p = new Person("Lacava", "Alessandro", 1976)
  val p = Person("Lacava", "Alessandro", 1976)
  ```

- 为参数列表中的所有参数添加`val`前缀，表示这些参数作为类的不可变成员，因此你会得到所有这个成员的*访问器*方法，而没有*修改器*方法：

  ```scala
  val lastname = p.lastname
  p.lastname = "Brown"		// 编译失败
  ```

- 为`hashCode`、`equals`、`toString`方法添加原生实现。因为`==`在 Scala 中代表`equals`，因此 case 类实例之间总是以**结构的方式**进行比较，即**比较数据而不是比较引用**：

  ```scala
  val p_1 = Person("Brown", "John", 1969)
  val p_2 = Person("Lacava", "Alessandro", 1976)

  p == p_1 // false
  p == p_2 // true
  ```

- 生成一个`copy`方法，使用现有的实例并接收一些新的字段值来创建一个新的实例：

  ```scala
  val p_3 = p.copy(firstname = "Michele", birthYear = 1972)
  ```

- 最重要的特性，**实现一个 `unapply` 方法**。因此，case 类可以支持模式匹配。这在定义 ADT 时尤为重要。`unapply`方法就是一个**析构器**。

- 当不需要参数列表时，可以定义为一个`case object`

## 常用其他特性

- 创建一个函数，根据提供的参数创建一个 case 类的实例：

  ```scala
  val personCreator: (String, String, Int) => Person = Person.apply _
  personCreator("Brown", "John", 1969)	// Person(Brown,John,1969)
  ```

- 如果需要将上面的函数**柯里化**，**分多步**提供参数来创建一个实例：

  ```scala
  val curriedPerson: String => String => Int => Person = Person.curried

  val lacavaBuilder: String => Int => Person = curriedPerson("Lacava")

  val me = lacavaBuilder("Alessandro")(1976)
  val myBrother = lacavaBuilder("Michele")(1972)
  ```

- 通过一个元组来创建实例：

  ```scala
  val tupledPerson: ((String, String, Int)) => Person = Person.tupled

  val meAsTuple: (String, String, Int) = ("Lacava", "Alessandro", 1976)

  val meAsPersonAgain: Person = tupledPerson(meAsTuple)
  ```

- 将一个实例转换成一个由其参数构造的元组的`Option`：

  ```scala
  val toOptionOfTuple: Person => Option[(String, String, Int)] = Person.unapply _

  val x: Option[(String, String, Int)] = toOptionOfTuple(p) // Some((Lacava,Alessandro,1976))
  ```

`curried`和`tupled`方法通过伴生对象继承自`AbstractFunctionN`。`N`是参数的数量，如果`N = 1`则并不会得到这两个方法。

### 以 *柯里化* 的方式定义 case 类

[Scala Case Classes In Depth](http://www.alessandrolacava.com/blog/scala-case-classes-in-depth/)

### 其他内建方法

因为所有的 case 类都会扩展`Product`特质，因此他们会得到如下方法：

- `def productArity:Int`：获得参数的数量
- `def productElement(n:Int):Any`：从 0 开始，获得第 n 个参数的值
- `def productIterator: Iterator[Any]`：获得由所有参数构造的迭代器
- `def productPrefix: String`：获得派生类中用于`toString`方法的字符串，这里会返回类名

