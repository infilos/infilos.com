---
type: docs
title: "函数式-基础"
linkTitle: "函数式-基础"
weight: 2
---

## 方法

创建函数最简单的方式就是作为对象的成员，即**方法**。

## 本地函数

函数式编程风格的一个主要原则就是：程序应该被解构为若干小的函数块，每块实现一个完备的任务，每块都很小。

但是大量的小块函数会污染程序的命名空间，这时可以私有函数，或者另一种方式，**本地函数**，即嵌套函数：

```scala
import scala.io.Source
object LongLines {
  def processFile(filename:String, width:Int){
    def processLines(line:String){    // 打印超出宽度的行, 引用外层的 width
      if (line.length > width) println(filename + ": " + line)
  }
  val source = Source.fromFile(filename)
  for (line <- source.getLines)
    processLine(line)
  }
}
```

## 头等函数

Scala 的函数是头等函数(first-class-function)，不仅可以定义或调用函数，还可以把函数写成匿名字面量，当做值来传递。函数字面量被编译进类，并在运行期实例化为函数值。因此函数字面量与值的区别在于：函数字面量存在于源代码，而函数值作为对象存在于运行期。类似于类(源代码)和对象(运行期)之间的关系。

```scala
(x: Int) => x + 1
```

符号`=>`指：把左边的东西转换为右边的东西。

函数值是对象，因此可以赋给变量。同时也是函数，可以按照通常的函数调用方式，使用一个括号来调用：

```scala
var increase = (x: Int) => x + 1
increase(2)   // 3
```

函数可以由多条语句构成，由大括号包围，组成一个代码块。当函数执行时，所有的语句都会执行，最后一行作为返回值。

## 函数字面量的短格式

可以省略函数字面量中的参数类型来进行简化：

```scala
someNumber.filter((x) => x > 0)
```

因为使用这个函数来过滤整数列表`someNumber`，因此 Scala 可以推断出 x 肯定为整数。这种方式称为**目标类型化(target typing)**。

某些参数的类型是推断的，可以省略参数的括号：

```scala
someNumber.filter(x => x > 0)
```

## 占位符语法

只要每个参数在函数字面量中仅出现一次，可以使用占位符`_`来代替该参数名：

```scala
someNumber.filter(_ > 0)
```

但是有时候把下划线当做参数的占位符，编译器可能无法推断缺失的参数类型：

```scala
val f = _ + _
```

因为上面的`filter`调用是一个整数列表，编译器可以推断，但这里，编译器无从推断，会编译失败。这时，我们可以为参数提供类型：

```scala
val f = (_:Int) + (_:Int)
```

这里需要注意的是，这里的两个占位符表示的是需要两个参数，而不是一个参数使用两次。

```scala
_ + _   (x, y) => x + y
_ * 2     x => x * 2
_.head    xs => xs.head
_ drop _  (xs, n) => xs.drop(n)
```

在参数较少时，使用这种方式可以使程序更清晰，但是当参数增多，比如`foo(_, g(List(_ + 1), _))`，则会让程序变得难以理解。

## 偏应用函数

或称为部分应用函数(Partially applied functions)。

可以使用占位符代替一个参数，也可以代替整个参数列表。比如`println(_)`，或者`println _`。

这中下划线方式的使用，实际上定义的是一个**部分应用函数**。它是一种表达式，不需要提供函数需要的所有参数，可以只提供部分，或者不提供。

比如一个普通的`sum`函数：

```scala
def sum(a:Int, b:Int, c:Int) = a + b + c
```

当调用函数时，传入任何需要的参数，实际是把函数应用到参数上。

如果通过`sum`来创建一个部分应用表达式，不需要提供所需要的参数，只需要在`sum`之后添加一个下划线，使用空格隔开，然后把得到的函数存入变量：

```scala
val a = sum _
a : (Int, Int, Int) => Int = <function3>  // function3 类型的实例
a(1,3,4)  // 6
```

可以看到，`a`为一个函数。变量`a`指向一个函数值对象。这个函数值是由 Scala 编译器依照部分应用函数表达式`sum _`自动产生的类的一个实例。编译器产生的类有一个`apply`方法，该方法接收 3 个参数。在通过`a`进行`a(1,2,3)`调用时，会被翻译成对函数值的`apply`方法的调用，即`apply(1,2,3)`。

可以看到`function3`的源码定义：

```scala
/** A function of 3 parameters.
 *
 */
trait Function3[-T1, -T2, -T3, +R] extends AnyRef { self =>
  /** Apply the body of this function to the arguments.
   *  @return   the result of function application.
   */
  def apply(v1: T1, v2: T2, v3: T3): R
  ...
}
```

偏应用函数实际上是指该函数未被应用到所有的参数，`sum`的例子是没有应用到任何参数，也可以只应用到部分参数。

```scala
val b = sum(1, _:Int, 3)
b: Int => Int = <function1>
```

这时会发现`b`为一个`function1`类型的实例，它接收一个参数。

同时，如果定义的是一个省略所有参数的偏应用函数，比如这里的`sum`，二者在代码的某个位置需要一个相同签名的函数，这时可以省略掉占位符进行简化：

```scala
someNumbers.foreach(println _)
someNumbers.foreach(println)
```

需要注意的是，只有在需要一个函数的地方才可以省略占位符。比如这里，编译器知道`foreach`需要一个函数。否则会编译错误。比如：`val s = sum`，必须是：`val s = sum _`。

## 闭包

## 重复参数

可以指定一个函数的最后一个参数是重复的，然后就可以传入可变长度的参数列表：

```scala
def echo(args:String*) = 
  for(arg <- args) println(arg)

echo("a")
echo("a", "b")
val seq = Seq("a","b","c")
echo(seq:_*)
```

## 尾递归

## 高阶函数

**函数值**作为参数的函数称为**高阶函数**。

### 减少代码重复

比如我们要实现一个 根据文件名查找文件的程序，首先是**文件名以指定字符串结尾的文件名**：

```scala
object FileMatcher { 
  private def filesHere = (new java.io.File(".")).listFiles
  def filesEnding(query: String) = 
    for (file <- filesHere; if file.getName.endsWith(query)) yield file
}
```

`filesHere`这里作为一个工具函数来获取所有文件名列表。

现在如果需要的不只是以指定字符串结尾的方式查找，只要是**文件名中包含指定字符串**，或者**以指定的方式能够匹配指定的字符串**，因此我们需要查找的方式是这样的：

```scala
def filesMatching(query: String, method ) = 
  for (file <- filesHere; if file.getName.method (query)) yield file
```

`method`表示匹配方式，但是 Scala 中不支持传入函数名的方式，因此我们可以传递一个函数值：

```scala
def filesMatching(query: String, matcher: (String, String) => Boolean) = {
  for (file <- filesHere; if matcher(file.getName, query)) yield file
}
```

`mathcer`接收两个字符串，一个是文件名，一个是需要匹配的字符串，返回一个布尔值表示该文件名与指定的字符串是否匹配。因此，我们可以实现我们的不同匹配方式，而对`filesMatching`函数进行复用：

```scala
def filesEnding(query: String) = filesMatching(query, _.endsWith(_))
def filesContaining(query: String) = filesMatching(query, _.contains(_))
def filesRegex(query: String) = filesMatching(query, _.matches(_))
```

类似`_.endsWith(_)`的部分使用了占位符语法，前面已经提到，函数的参数只被使用一次，且参数顺序与使用顺序一致，则可以使用占位符语法简化。其完整的写法实际是：

```scala
(fileName: String, query: String) => fileName.endsWith(query)
```

简化后会发现，函数`filesMatching`的参数中，`query`已经不再需要了，因为该参数只用于`matcher`函数，并且已经通过匹配方法传入，因此再次进行简化：

```scala
object FileMatcher{
  private def filesHere = (new java.io.File(".")).listFiles
  private def filesMatching(matcher: String => Boolean) = 
    for (file <- filesHere; if matcher(file.getName)) yield file
    
  def filesEnding(query:String) = filesMatching(_.endsWith(query))
  def filesContains(query:String) = filesMatching(_.contains(query))
  def filesRegex(query:String) = filesMatching(_.matches(query))
}
```

### 简化客户端代码

集合 API 提供一些列常用的方法，其中应用了大量的高阶函数，通过将高阶函数作为参数来定义 API，从而使客户端代码更加易于使用。

比如常用的`exists`、`find`， 在`scala.collection.TraversableLike`包中：

```scala
def exists(p: A => Boolean): Boolean = {
  var result = false
    breakable {
      for (x <- this)
        if (p(x)) { result = true; break }
    }
    result
}

def find(p: A => Boolean): Option[A] = {
    var result: Option[A] = None
    breakable {
      for (x <- this)
        if (p(x)) { result = Some(x); break }
    }
    result
}
```

参数`p`是一个`A => Boolean`类型的函数，它接收一个参数并放回一个布尔值，用于判断集合中的元素是否满足条件，比如在应用时：

```scala
List(1,2,3,-1).exists(_ < 0)
```

这里同样应用了占位符语法，使整个操作更加简便。

## 柯里化

柯里化是将函数应用到**多个参数列表上**。

```scala
def plainOldSum(x: Int, y: Int) = x + y   // 普通函数
plainOldSum(1, 2)
def curriedSum(x: Int)(y: Int) = x + y    // 柯里化函数
curriedSum(1)(2)
```

在调用柯里化函数时，如果没有一次给出所有的参数列表，比如上面的`curriedSum`，第一次只提供一个参数列表进行调用`curriedSum(1)`，这时会返回一个函数值`(y: Int) => x + y`，调用该函数值并提供另一个参数列表，即`(y: Int)`，得出最后的求和值。

其过程类似于以下面的方式定义函数：

```scala
def first(x: Int) = (y: Int) => x + y
val second = first(1) // Int => Int = <function1>
second(2)       // 3
```

柯里化函数也可以以下面的方式，使用一个占位符语法，来获取中间的函数值，即上面的`second`函数值：

```scala
val onePlus = curriedSum(1)_    // Int => Int = <function1>
```

调用时提供的占位符`_`代表第二个参数列表。

同样，可以定义更多个参数列表的柯里化函数，比如：

```scala
def multiSum(x: Int)(y: Int)(z: Int) = x + y + z
val second = multiSum(1)_   // Int => (Int => Int) = <function1>
val third = second(2)     // Int => Int = <function1>
third(3)            // 6
```

## 自定义控制结构

因为函数可以作为参数值来传递，因此可以使用该特性来定义自己的控制结构，只需要定义接收函数值的方法即可。

比如：

```scala
def twice(op: Double => Double, x: Double) = op(op(x))
twice(_ + 1, 5)   // 7
```

一旦发现代码中有重复的控制模式，就可以通过定义一个函数的方式来代替。

比如我们需要一个控制结构，**操作一个文件并最终将其关闭**，这就是一个控制模式，而**操作**部分是主要的处理：

```scala
def withPrintWriter(file: File, op: PrintWriter => Unit) = { 
  val writer = new PrintWriter(file) 
  try {
    op(writer) 
  } finally {
    writer.close() 
  }
}

withPrintWriter( 
  new File("date.txt"), 
  writer => writer.println(new java.util.Date) 
)
```

我们可以利用这种模式来实现不同的控制，比如将`withPrintWriter`实现为一个日志打印过程或缓存更新过程，而操作部分是一次数据库查询。

这种模式成为**借贷模式**。这个例子中，控制抽象函数`withPrintWriter`打开一个资源，即`writer`，借贷给`op`函数，当`op`函数不再需要时又将其关闭。

但是这种模式的使用方式扛起来并不像是一个控制结构，它就是一个函数调用，然而可以通过使用大括号的方式使其更像是真正的控制结构。

但是在使用大括号进行函数调用时只能接收一个参数，比如：

```scala
println { "Hello, world!" }
"Hello, world!".substring { 7, 9 }    // error，多个参数必须使用小括号包围参数列表
```

因此，我们可以将上面的`withPrintWriter`函数改写为柯里化的方式，每次只接收一个函数，来满足只有一个参数才能使用大括号的要求：

```scala
def withPrintWriter(file: File)(op: PrintWriter => Unit) = { 
  val writer = new PrintWriter(file) 
  try {
    op(writer) 
  } finally {
    writer.close() 
  }
}
```

然后，下面的调用使`withPrintWriter` 看起来更像一个控制结构：

```scala
withPrintWriter(file) { writer =>
  writer.println(new java.util.Date) 
}
```

## 传名参数

但是上面的例子与内建的控制接口，比如`if`、`while`并不相似，因为在大括号中需要传入一个参数，这可以通过**传名参数**实现。

比如定义一个断言函数：

```scala
var assertionsEnabled = true

def myAssert(predicate: () => Boolean) = 
  if (assertionsEnabled && !predicate()) throw new AssertionError
```

然后以下面的方式调用：

```scala
myAssert(() => 5 > 3)
```

或许你更希望使用`myAssert(5 > 3)`的方式来调用，在创建传名参数时可以使用`=>`来代替完整的`() =>`，括号部分实际是函数的参数列表，只不过该函数不接受任何参数，因此省略。

```scala
def byNameAssert(predicate: => Boolean) = 
  if (assertionsEnabled && !predicate) throw new AssertionError

myAssert(5 > 3)
```

一个传名类型，即其参数列表为空`()`，并进行省略，这样的用法仅仅在作为参数时可行。

但是根据上面的定义方式，其实与`def byNameAssert(predicate:Boolean)`没有什么差别了。

真正的差别在于，如果是传入的值，这个值必须在调用`byNameAssert`之前完成计算，如果是传入的函数，则会在调用之后进行计算。