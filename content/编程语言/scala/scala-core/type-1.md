---
type: docs
title: "Type-用例"
linkTitle: "Type-用例"
weight: 13
---

## 介绍

Scala 有一个强大的类型系统。然而，除非你是一个库的创建者，你可以不用深入了解类型系统。但是一旦你要为其他用户创建集合类型的 API，你就需要学习这些。

Scala 类型系统使用一组标示符来表示不同的泛型类型概念，包括**型变、界限、限定**。

### 界限

界限(bound)用于限制类型参数。界限标示符汇总：

| 标示符                 | 名称      | 描述          |
| ------------------- | ------- | ----------- |
| A <: B              | 上界      | A 必须是 B 的子类 |
| A >: B              | 下界      | A 必须是 B 的父类 |
| A <: Upper >: Lower | 同时使用上下界 | A 同时用于上界和下界 |
| A <% B              | 视图界限    |             |
| T : M               | 上下文界限   |             |

### 类型限定

Scala 允许你指定额外的类型限制：

```scala
A =:= B 	// A 和 B 必须相同
A <:< B 	// A 必须是 B 的子类型
A <%< B		// A 必须是 B 的视图类型
```

### 常用类型参数标示符

| 标示符   | 说明                                 |
| ----- | ---------------------------------- |
| A     | 用于一个简单的类型时，`List[A]`                 |
| B,C,D | 用于同时需要多个类型时                        |
| K     | 在 Java 中常用于 Map 的 key，Scala 中多使用 A |
| N     | 用于一个数字类型                           |
| V     | 跟 V 类似，Scala 中多用 B                 |

## 类型参数化

类型参数化用于编写泛型类和特质。

## 实例：queues 函数式队列

函数式队列是拥有以下三种操作的数据结构：

```scala
head	返回队列的第一个元素
tail	返回除第一个元素之外的队列
append	返回尾部添加了指定元素的队列
```

**不同于可变队列，函数式队列在添加元素时不会改变其内容，而是返回包含这个元素的新队列。**

支持的工作方式：

```scala
val q1 = Queue(1,2,3)
val q2 = q1.append(4)
print(q1)	// Queue(1,2,3)
```

如果将 Queue 实现为可变类型，则 append 操作会改变 q1 的值，这是 q1 和 q2 都拥有新的元素 4。但是对于函数式队列来说，新添加的元素只能出现在 q1 中而不能出现在 q2 中。

纯函数式队列与 List 具有相似性，都被称为是**完全持久的数据结构**，即使在扩展或改变之后，旧的版本依然可用。但是 List 通常使用`::`操作在前端扩展，队列使用`append`在后端扩展。

Queue 需要保持三种操作都能在常量时间内完成，低效的实现和最后的实现：

```scala
class SlowAppendQueue[T](elems:List[T]){
  def head = elems.head
  def tail = new SlowAppendQueue(elems.tail)
  // append 操作的时间花费与元素的数量成正比
  def append(x:T) = new SlowAppendQueue(elems :: List(x))
}

class SlowHeadQueue[T](smele:List[T]){
  // 将 elems 元素顺序翻转
  def head = smele.last								// 与元素个数成正比
  def tail = new SlowHeadQueue(smele.init)			// 与元素个数成正比
  def append(x:T) = new SlowHeadQueue(x :: smele)	// 常量
}

class Queue[T](
  private val leading: Lit[T]
  private val trailing: List[T]
) {
  private def mirror = {
    if (leading.isEmpty) new Queue(trailing.reverse :: Nil)
    else this
  }
  
  def head  = mirror.leading.head
    
  def tail = {
  	val q = mirror
    new Queue(q.leading.tail, q.trailing)
  }
  
  def append(x: T) = new Queue(leading, x :: trailing)
}
```

最终的方案中使用两个 List，leading 和 trailing 来表达队列。leading 包含前段元素，trailing 包含了反向排序的后段元素，整个队列表示为：`leading :: trailing.reverse`。

- append 时，使用`::`将元素添加到 trailing，时间为常量。如果一直通过添加操作构建队列，则 leading 部分会一直为空。
- tail 时，如果 tailing 不为空会直接返回。如果为空，需要将 tailing 翻转并复制给 leading 然后取第一个元素返回，这个操作称为 mirror，时间与元素量成正比。
- head 与 tail 类似。

这种方案基于三种操作的频率接近。

## 信息隐藏

上面 Queue 的实现中暴露了细节实现。

### 私有构造器及工厂方法

**使用私有构造器和私有成员来隐藏类的初始化代码和表达代码。**

Java 中可以把主构造器声明为私有使其不可见，Scala 中无须明确定义。虽然它的定义可以隐含在类参数以及类方法体中，还是可以通过在类参数列表前添加 private 修饰符把主构造器隐藏起来：

```scala
class Queue[T] private (
  private val leading:List[T],
  private val trailing:List[T]
)
```

这个参数列表之前的 private 修饰符表示 Queue的构造器是私有的：它只能被类本身或伴生对象访问。类名 Queue 仍然是公开的，因此可以继续使用这个类，但不能调用它的构造器。

可以通过添加辅助构造器来创建实例：

```scala
def this() = this(Nil, Nil)	 					// 可以构建空队列
def this(elems:T*) = this(elems.toList, Nil)	// 可以提供队列初始元素
```

另一种方式添加一个工厂方法，最简单的方式是定义与类同名的对象及 apply 方法：

```scala
object Queue{
  def apply[T](xs:T*) = new Queue[T](xs.toList, Nil)
}

// Usage
Queue(1，2，3)
```

同时将该对象放到 Queue 类同一个源文件，成为 Queue 类的伴生对象。

### 供选方案：私有类

**直接将类隐藏掉，仅提供能够暴露类公共接口的特质。**

```scala
trait Queue[T]{
  def head:T
  def tail:Queue
  def appand(x:T):Queue[T]
}

object Queue{
  def apply[T](xs:T*):Queue[T] = New QueueImpl[T](xs.toList, Nil)
  
  private class QueueImpl[T](
    private val leading:List[T],
    private val trailing:List[t]
  ) extends Queue[T]{
    def mirror = {
  	  if (leading.isEmpty) new QueueImpl(trailing.reverse, Nil) else this
	}
	
	def head:T = mirror.leading.head
	
	def tail:QueueImpl[T] = {
      val q = mirror
      new QueueImpl(q.leading.tail, q.trailing)
	}
	
	def append(x:T) = new QueueImpl(leading, x :: trailing)
  }
}
```

## 型变注解

上面定义的 Queue 是一个特质而不是类型，因为他带有类型参数。即 Queue 是特质，也可称为类型构造器(给它提供参数来构建新的类型)，`Queue[Int]` 是类型。

**带有参数的类和特质是泛型的，但是他们产生的类型已被“参数化”，不再是泛型的。**

**泛型：指通过一个能够广泛适用的类或特质来定义许多特定的类型。**

**型变：泛型话类型`(Queue[T]`)产生的类型家族成员(`Queue[String]`,`Queue[Int]`,...)之间具有的特定的子类型关系。**定义了参数化类型传入方法的规则。

**协变：如果 S 类型是 T 类型的子类型，同时 `Queue[S]` 也是 `Queue[T]` 的子类型，即认为，Queue 是与他的类型参数 T 保持协变的。**

> 协变意味着，如果S 为 T 的子类型，向能够接受参数类型为 `Queue[T]` 的函数传入类型为 `Queue[S]` 的参数。比如方法签名`def func(arg:Queue[AnyRef])`,可以调用`func(Queue[String])`，因为 String 是 AnyRef 的子类型。

**非型变：Scala 中默认的泛型类型是非型变的。即不泛型类型产生的类型家族成员之间没有子类型关系。**

> 非型变意味着，即便是类型参数之间有子类型关系，比如 String 是 AnyRef 的子类型，但是泛型类型为非型变，则 `Queue[String]` 不能当做 `Queue[AnyRef]` 来使用，必须使用定义的 `Queue[AnyRef]` 类型。

**逆变：如果 T 是 S 的子类型，表示 `Queue[T]` 是 `Queue[S]` 的父类型。**

> 逆变意味着，如果 S 为 T 的子类型，向能够接受参数类型为 `Queue[S]` 的函数传入类型为 `Queue[T]` 的参数。

**这里说的参数传入的例子，即：方法参数预期为父类，传入必须为子类(里氏替换原则：任何使用父类的地方都可以用子类替换掉，因为子类拥有父类所有的属性和方法，即只需要一个父类就完成的工作，传入了一个功能更多的子类当然也能完成需要的工作)。**

### 型变参数：

| 标示符               | 名称     | 描述                                       |
| ----------------- | ------ | ---------------------------------------- |
| `Array[T]`          | 非型变类型 | 当容器中的元素是可变的即 collections.mutable。比如：预期参数为 `Array[String]` 的方法只能传入 `Array[String]` |
| `Seq[+A]`           | 协变类型   | 当容器中的元素是不可变的，这使容器更灵活。比如，预期参数为 `Seq[Any]` 的方法可以传入 `Seq[String]` |
| `Foo[-A]`           | 逆变类型   | 与协变相反                                    |
| `Function1[-A, +B]` | 组合型变   | 参考 Function1 特质的定义                       |

一个型变的实例：

```scala
// 一组具体类型
class Grandparent 
class Parent extends Grandparent 
class Child extends Parent

// 一组容器类型
class InvariantClass[A] 		// 不变容器类型，容器中只能传入类型 A
class CovariantClass[+A] 		// 协变容器类型，容器中只能传入类型 A 和 A 的子类型 
class ContravariantClass[-A]	// 逆变容器类型，容器中只能传入类型 A 和 A 的父类型

class VarianceExamples {

  def invarMethod(x: InvariantClass[Parent]) {}
  def covarMethod(x: CovariantClass[Parent]) {}
  def contraMethod(x: ContravariantClass[Parent]) {}

  invarMethod(new InvariantClass[Child]) 				// 正确
  invarMethod(new InvariantClass[Parent]) 				// 错误
  invarMethod(new InvariantClass[Grandparent])			// 错误

  covarMethod(new CovariantClass[Child]) 				// 正确
  covarMethod(new CovariantClass[Parent]) 				// 正确
  covarMethod(new CovariantClass[Grandparent])			// 错误

  contraMethod(new ContravariantClass[Child]) 			// 错误
  contraMethod(new ContravariantClass[Parent]) 			// 正确
  contraMethod(new ContravariantClass[Grandparent])		// 正确
}
```

一个逆变的例子：

```scala
trait OutputChannel[-T] {
  def write(x: T)
}
```

这里定义 OutputChannel 是逆变的，比如：一个 `Channel[AnyRef]` 会是 `Channel[String]` 的子类型。如果用做一个方法参数：`def func(arg: Channel[String])`,可以调用为：`func(Channel[AnyRef])`。

### 型变与数组

Scala 认为 数组是**非型变**的。

### 检查型变注解

只要泛型的参数类型被当做方法参数的类型，那么包含它的类或特质就有可能不与类型参数一起协变。

比如：

```scala
class Queue[+T] {
  def append(x: T) ...
}
```

类型 T 即作为泛型 Queue 的参数类型，又作为方法 append 的参数类型，这是不允许的，编译器会报错。

## 下界

上面例子中 `Queue[T]` 不能实现对 T 的协变，因为 T 作为参数类型出现在了 append 方法中。想要解决这个问题，可以把 append 变为多态使其泛型化，并使用它的类型参数的下界：

```scala
class Queue[+T](...){
  def append[U >: T](x: U) = new Queue[U](leading, x :: trailing) ...
}
```

这个定义指定了 append 的类型参数 U，并通过语法`U >: T`定义了 T 为 U 的下界，即 U 必须是 T 的超类。

> 比如类 Fruit 及两个子类 Apple、Orange。现在可以吧 Orange 对象加入到 `Queue[Apple]`，返回个 `Queue[Fruit]` 类型。

这个定义支持，队列类型元素类型为 T，即`Queue[T]`,允许将任意 T 的超类 U 的对象加入到队列中，结果为 `Queue[U]`。

## 对象私有数据

为了避免 leading 一直为空导致的 mirror 不断重复的执行，下面是改进后的 Queue 定义：

```scala
class Queue[+T] private (
  private[this] var leading: List[T],
  private[this] var trialing: List[T]
) {
  private def mirror() = {
    if(leading.isEmpty) {
	  while(!trailing.isEmpty) {
  		leading = trailing.head :: leading
        trailing = trailing.tail
	  }  
    }
  }
  
  def head: T = {
    mirror()
    leading.head
  }
  
  def tail: Queue[T] = {
    mirror()
    new Queue(leading.tail, trailing)
  }
  
  def append[U >: T](x: U) = new Queue[U](leading, x :: trailing)
}
```

这个版本中的 leading 和 trailing 都是可变变量。而 mirror 从 trailing 反向复制到 leading 的操作通过副作用对两段队列进行修改而不是返回队列。由于二者都是私有变量，因此这些操作对客户端是不可见的。

同时，leading 和 trailing 都被 `private[this]`修饰符声明对对象私有了，因此能通过类型检查。Scala 的型变检查包含了关于对象私有定义的特例。当检查到带有`+/-`符号的类型参数只出现在具有相同型变分类的位置上时，这种定义将被忽略。

## 上界

下面是一个为自定义类实现排序的例子。通过把 Ordered 混入类中，并实现 Ordered 唯一的抽象方法 compare，就可以使用 `<,>.<=,>=`来做类实例的比较：

```scala
class Person(val firstName:String, val lastName:String) extends Ordered[Person] {
  def compare(that:Person) = {
    val lastNameComparison = lastNmae.compareToIngoreCase(that.lastName)
    if (lastName.comparison != 0) lastNameComparison
    else firstName.conpareToIngoreCase(that.firstName)
  }
  
  override def toString = firstName + " " + lastName
}
```

为了让传递给你的新排序函数的列表类型混入到 Ordered 中，需要使用到上界。通过指定`T <: Ordered[T]`，表示类型参数 T 具有上界 `Ordered[T]`，即传递给排序方法 orderedMergeSort 的列表元素类型必须是 Ordered 的子类型。因此，可以传递 `List[Person]` 给该方法，因为 Person 混入了 Ordered。

```scala
def orderedMergeSort[T <: Ordered[T]](xs: List[T]): List[T] = {
  def merge(xs: List[T], ys: List[T]): List[T] = {
    (xs, ys) match{
      case (Nil,_) => ys
      case (_, Nil) => xs
      case (x :: xsl, y: ysl) =>
        if (x < y) x :: merge(xsl, ysl)
        else y :: merge(xs, ysl)
    }
  }
  
  val n = xs.length / 2
  if (n == 0) xs
  else {
    val (ys, zs) = xs splitAt n
    merge(orderedMergeSort(ys), orderedMergeSort(zs))
  }
}
```

## 实例

### 如何使用泛型类型创建类

创建一个能够接受泛型类型的类或方法，比如创建一个链表类：

```scala
class LinkedList[A] {
  private class Node[A](elem:A){
    var next: Node[A] = _
    overrice def toString = elem.toString
  }
  
  private var head:Node[A] = _
  
  def add(elem:A){
    val n = new Node(elem)
    n.next = head
    head = n
  }
  
  private def printNodes(n:Node[A]) = {
    if (n != null){
      println(n)
      printNoeds(n.next)
	}
  }
  
  def printAll(){ printNodes(head) }
}
```

`[A]`是该类的参数化类型，要创建一个 Int 类型的链表实例：`val ints = new LinkedList[Int]()`,

此时这个链表的整体类型为`LinkedList[Int]`，可以向其添加 Int 类型的节点：`ints.add(1)`。

或者创建其他类型的链表：`val strings = new LinkedList[String]`或`val foos = new LinkedList[Foo]`。

当创建一个基本类型的泛型实例时，比如：`val anys = new LinkedList[Any]`，这是可以传入基本类型 Any 的子类型比如 Int，`anys.add(1)`。但是如果有一个方法：

```scala
def printTypes(elems:LinkedList[Any]) = elems.printAll()
```

这时并不能传入一个`ListedList[Int]`到该方法，这需要这个链表直接**协变**。

如果同时需要多个类型参数，比如：

```scala
trait Pair[A, B]{
  def getKey:A
  def getValue:B
}
```

### 如何使用泛型类型创建方法

创建一个带有类型参数的方法能够使其用于更多的适用范围：

```scala
def randomElement[A](seq:Seq[A]):A = {
  val randomNum = util.Random
  seq(randomNum)
}
```

### 如何使用鸭子类型(结构化类型)

```scala
def callSpeak[A <: { def speak(): Unit }](obj:A){
  obj.speak()
}
```

在这个定义中，方法`callSpeak`可以接受任意一种类型 A，只要该类型拥有一个类型参数中定义签名的 speak 方法。

类型参数语法`[A <: { def speak(): Unit }]`表示，类型 A 必须是一个拥有方法`def speak(): Unit`的类型的子类型，即上界语法。同时需要注意的是，这个父类中的方法的签名必须与类型参数中定义的签名一致。

### 使可变集合非型变

在定义一个元素可变的集合时，其元素类型必须是非型变的，即`[A]`。

使用非型变类型会有一些副作用，比如，容器可以同时接收基本类型或其子类型。同时，如果一个方法被声明为接收一个父类型的容器，比如`ArrayBuffer[Any]`，传入`ArrayBuffer[Int]`则不会通过编译。因为：

- ArrayBuffer 中的元素是可以改变的
- 定义的方法接收的是`ArrayBuffer[Any]`，但传入的却是`ArrayBuffer[Int]`
- 如果编译器通过了，集合会使用 Any 代替普通的 Int 类型，这是不允许的

如果想要一个方法技能接收父类型的集合，又能接收其子类型的集合，需要使用一个不可变的集合类型，比如 List、Set 等。

在 Scala 中，可变集合是非型变的，而不可变集合为协变，参考协变与飞行变的区别。

### 使不可变集合协变

正如**协变**中的说明一样，不可变集合被定义为协变，则，使用这类集合作为参数的方法，同样能够接受其子类型的集合作为参数。

创建一个不可变容器，并声明其为协变：

```scala
class Container[+A] (val emel:A)

def makeDogsSpeak(dogHouse:Container[Dog]){
  dogHouse.elem.speak()
}

makeDogsSpeak(new Container(Dog("dog of Dog")))
// SubDog is sub type of Dog
makeDogsSpeak(new Container(SubDog("dog of SubDog"))) 
```

### 限制类型参数的范围

在一个拥有类型参数的类或方法中，如果需要限制该类型参数的范围，可以使用**上界**或**下界**来限制类型参数的可选范围。

比如有一些多重继承的类：

```scala
class Professor()
class Teacher() extends Professor
class Student()
class Child() extends Student
```

假设一些场景：

```scala
def teach[A](A >: Teacher)
def learn[A](A <: Student)
```

这里，只有老师或教授能够讲课，即下界，最少为老师。只有学生或小孩能够学习，即上界。

### 选择性的为封闭模型添加新行为

比如想要给所有的数字类型增加一个求和方法，比如`Int、Double、Float`等。因为`Numeric`类型类已经存在，这支持你创建一个能够接受一个任意数字类型的求和方法：

```scala
def add[A](x:A, y:A)(implicit  numeric: Numeric[A]):A = numeric.plus(x,y)
```

然后，这个方法就可以用于不同的数字类型求和：

```scala
add(1, 3)
add(1.0, 1.5)
add(1, 1.5f)
```

#### 如何创建一个类型类(type class)

创建类型类的过程有点复杂，单仍然可以总结为一个公式：

- 通常你有一个需求，为一个封闭的模型增加新的行为
- 为了增加这个行为，你会创建一个类型类。通常的方式是，创建一个基本的特质，然后使用隐式对象对该特质创建具体的实现
- 然后回到应用中，创建一个使用该类型类的方法将新的行为添加到封闭模型，比如上面创建的 add 方法

比如你有一些封闭模型，包含一个 Dog 和 Cat，你想要 Dog 能够说话而 Cat 不能。

首先是封闭模型：

```scala
// 一个已存在的封闭模型
trait Animal
final case class Dog(name:String) extends Animal
final case class Cat(name:String) extends Animal
```

为了能够给 Dog 添加说话方法，创建一个类型类并为 Dog 实现 speak 方法：

```scala
object Humanish{
  // 类型类，创建一个 speak 抽象方法
  trait HumanLike[A]{
    def speak(speaker:A): Unit
  }
  
  // 伴生对象
  object HumanLike{
    // 为需要的类型实现要增加的行为，这里只要为 Dog 实现
    implicit object DogIsHumanLike extends HumanLike[Dog]{
      def speak(dog:Dog){ println("I'm a dog, my name is ${dog.name}") }
    }
  }
}
```

定义完新的行为后，在应用中使用该功能：

```scala
object TypeClassDemo extends App{
  // 创建一个方法能够使动物说话
  def makeHumanLikeThingSpeak[A](animal:A)(implicit humanLike: HumanLike[A]){
    humanLike.speak(animal)
  }
  
  // 因为 HumanLike 中实现了 Dog 类型的方法，因此可以用于 Dog 类型
  makeHumanLikeThingSpeak(Dog("Rover"))
  
  // 但是 HumanLike 中并没有 Cat 类型的实现，因此不能用于 Cat 类型
  // makeHumanLikeThingSpeak(Cat("Mimi"))
}
```

这里需要注意的是：

- 方法 makeHumanLikeThingSpeak 类似于本节开头的 add 方法
- 因为 Numeric 类型类已经由 Scala 定义，因此可以自己用来创建自己的方法。否则，需要创建自己的类型类，这里就是 HumanLike 特质
- 因为 speak 方法定义于 DogsIsHumanLike 中，该隐式对象继承于 `HumanLike[Dog]`，因此只能将一个 Dog 对象传入 makeHumanLikeThingSpeak 方法，而不能是一个 Cat 对象

> 这里的 类(class) 概念并不是来自面向对象的世界，而是函数式编程的世界。正如上面例子中演示的，一个 类型类(type class) 的益处在于能为一个已存在的(不能再进行修改的)封闭模型添加新的行为。另一个益处在于，能够为泛型类型创建方法，并且能够控制这些泛型类型，比如只有 Dog 可以说话。

### 与类型构建功能

#### 创建一个计时器

在 Unix 系统中，可以使用一下命令来查看一个执行过程花费的时间：

```shell
time find . -name "*.scala"
```

在 Scala 中我们可以创建一个类似的方法来查看对应执行过程消耗的时间：

```scala
val (result, time) = timer(someLongRunningAlgorithm)
println(s"result: $result, time: $time")
```

这个方法中，timer 方法会执行传入的`someLongRunningAlgorithm`方法，返回执行结果和其消耗的时间。

下面是 timer 的实现：

```scala
def timer[A](blockOfCode: => A) = {
  val startTime = System.nanoTime
  val result = blockOfCode
  val stopTime = System.nanoTime
  val delta = stopTime - startTime
  (result, delta/10000000d)
}
```

timer 方法使用**按名调用(call-by-name)**的语法来接收一个代码块作为一个参数。同时声明一个泛型类型最为该代码块的返回值，而不是指定声明为一个具体的类型比如 Int。这支持你传入任意类型的方法，比如：`timer(println("nothing"))`。

#### 创建自己的 Try 类

在 Scala 2.10 之前并没有 Try、Succeeded、Failed 这些类，如何自己实现以拥有以下的功能呢：

```scala
val x = Attempt("10".toInt)		// Succeeded(10)
val y = Attempt("10A".toInt)	// Failed(Exception)
```

首先需要实现一个 Attempt 类，同时为了不使用 new 关键字来创建实例，需要实现一个 apply 方法。还需要定义 Succeeded 和 Failed 类，并继承 Attempt。下面是第一个版本的实现：

```scala
sealed class Attempt[A]

object Attempt {
  def apply[A](f: => A) = {
  	try{
      val result = f
      return Succeeded(result)
    } catch {
      case e:Excaption => Failed(e)
    }
  }
}

final case class Failed[A](val exception:Throwable) extends Attempt[A]
final case class Succeeded[A](value:A) extends Attempt[A]
```

与上面的 timer 实现类似，apply 方法接收一个**按名调用**的参数，同时返回值是一个泛型类型。但是为了使这个类真正有用，还需要实现一个 getOrElse 方法来获取结果的信息，无论是 Failed 还是 Succeeded。

```scala
val x = Attempt(1/0)
val result = x.getOrElse(0)

val y = Attempt("foo".toInt).getOrElse(0)
```

下面我们实现这个 getOrElse 方法：

```scala
sealed abstract class Attempt[A]{
  def getOrElse[B >: A](default: => B): B = if (isSuccess) get else default
  var isSuccess = false
  def get:A
}

object Attempt{
  def apply[A](f: => A):Attempt[A] = 
    try{
  	  val result = f
  	  Succeeded(result)
	} catch {
      case e:Exception => Failed(e)
	}
}

final case class Failed[A](val exception:Thorwable) extends Attempt[A] {
  isSuccess = false
  def get:A = thorw exception
}

fianl case class Succeeded[A](result:A) extends Attempt[A]{
  isSuccess = true
  def get = result
}
```

这里需要注意的是方法 getOrElse 的签名：

```scala
def getOrElse[B >: A](default: => B): B = if (isSuccess) get else default
```

它的类型签名`[B >: A]`使用了下界，同时返回值的类型为 B，即该方法的返回值类型必须是 A 或 A 的父类。因为，在预期一个返回值是父类的地方可以返回一个子类，因为对父类的需求其子类都能满足，但是如果预期返回值是一个子类，但是返回一个父类，对子类要比父类的多，父类并不能满足使用需要，比如子类有个新的方法而父类中并没有，这时候返回了一个父类，再去调用该新方法时将会报错。即任何使用父类的地方都可以使用其子类替换，反之则行不通。