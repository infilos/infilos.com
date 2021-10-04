---
type: docs
title: "战略Scala风格"
linkTitle: "战略Scala风格"
weight: 45
---

[Strategic Scala Style: Practical Type Safety](http://www.lihaoyi.com/post/StrategicScalaStylePracticalTypeSafety.html)一文的中文翻译，点击查看原文。

这篇文章探索了如何利用Scala的类型安全特性来避免在使用Scala编写程序时出错。

虽然Scala有一个编译器帮助你捕捉错误，或者称它为类型安全，实际上有一个全方位的方式让你能够编写更多或更少安全性的Scala。我们将会讨论多种方式让你把你的代码转变为更加安全的系列。我们将有意识地忽略那些绝对的证明和逻辑事物的理论方面，而更专注于实践的方式来使编译器帮你不做更多隐藏的BUG。

`Type Safety`涉及面很广。你需要投入整个生涯来学习Haskell的类型系统或Scala变成语言，并且需要花费另一个周期来学习Haskell运行时的的类型实现或Java虚拟机。这里将会忽略这两个内容。

取而代之，本文将会以实践的方式来介绍如何以“类型安全”的方式来使用Scala，编译器的知识可以让你减轻错误的后果，并且能够把这些错误编程简单的、能够在开发期间完成修改的错误，以此来提高你代码的安全性。一个有经验的Scala开发者会发现本文的“基础“和”显而易见“，但是任何新手将会期望将这些技术添加到工具箱来解决Scala中遇到的问题。

这里的每一种技术描述都会做一些权衡：冗长、复杂、额外的类文件、低劣的运行时性能。本文我们会忽略这些问题而把他们当做是相当完美的。本文只会列举可能的情况，而不会去深入讨论有些取舍是否值得。而且，本文仅作用域纯粹的Scala，不会涉及类似Scalaz、Cats、Shapeless这样的第三方扩展库。如果属性该类技术的人愿意去写的话，这些库应该有他们自己的风格或技术并在他们自己的文章中展示。

## 原理

在我们讨论具体的技术和围绕类型安全的取舍之前，停下来思考一下问题本质是有意义的。什么是一个类型？”安全“一词有意味这什么？

### 什么是类型(Type)

**在一个程序的运行时，你对一个值的了解就是Type。**

基本上所有的编程语言都有一个不同的类型系统。一些有泛型，一些有具体化的泛型。例如Java，有具体化的泛型，一个值的类型总是与一个类护着接口符合并能在运行时检查。其他的，比如C就不能。Python这样的动态语言没有静态类型，因此类型仅存在于运行时。

本文所讲的Scala语言，拥有它自己的相对复杂的特定的类型系统。也有一些尝试将其正式化，比如[Dotty](https://github.com/lampepfl/dotty)项目。本文将会忽略这些。

本文中将会依照上面的释义。在一个程序的运行时，你对一个值的了解就是Type。比如：

1. 一个`Int`定义为包含了从`-2147483648`到`2147483647`的32位整数；
2. 一个`Option[T]`定义为要么是一个`Some[T]`，要么是一个`None`；
3. 一个`CharSequence`定义为包含了一些`Char`并支持我们调用`.length、.chatAt、.subsequence`等方法，但是我并不知道它是一个`String`、`String`或别的什么。你并不知道它是否可变(mutable/immutable)，它如何保存它的内容，或者性能规格如何；
4. 一个`String`同样拥有一些字符，但你知道它是不可变的，使用一个内部的字符数组来保存内容，通过索引来查找`Char`的时间复杂度我`O(1)`。

一个值的类似告诉你某些东西是什么和它不能是什么。`Option[String]`可能是`Some`或`None`，但它绝不会是一个32位的整数！在Scala中，这些是你不需要检查的：这些是你可以依赖的正确事物，编译器会在编译器为你检查。

这个知识点确切的指出了一个值的类型包含了什么。

### 类型不是什么

#### A Class

这个定义中，类型不是一个类(Class)。对，在基于JVM之上的Java和Scala，所有的类型被描述为类(class)或接口(interface)，这在`Scala.js`中并不有效，你可以定义一个假的类型(`trait`继承自`js.Any`)，在编译之后不会留下任何残留，或在其他编程语言中。

虽然类型被类所支持是一个事实，这只是一个实现细节并且与本位无关。

#### Scala类型系统

这里讨论的类型概念是含糊的、宽泛的，基于所有语言而不仅仅是Scala。Scala自身的类型系统是复杂的，有类，抽象类型，细化类型，特质，类型界限，上下文界限，和一些其他更加晦涩的东西。

纵观本文，这些细节都是为了服务一个目的：在你程序中将你对值的了解描述给编译器，然后让他检查你现在做的，与你说的和想要做的是否一致。

### 什么是安全

**类型安全意味着一点你出错，后果的影响比较小。**

相比类型，可能有其他更多对”安全“的定义。上面的定义比类型有宽泛：它作用于安全实践、隔离、分布式系统中的健壮性和恢复性，还有其他一些事情。

人们会犯各种错误：代码排版、可怜的负荷估算、复制粘贴错误的命令。当你出错时发生了什么？

* 你会看到编辑器中红色表示然后5s内修复了它；
* 你想完整的编译，花费了10s，然后修复了它；
* 你运行测试用例，花费了10s，然后修复了它；
* 你部署了这个错误，然后几个小时以后才发现它，然后修复后再部署；
* 你部署了这个错误，这个错误几周内都没有被提醒，但是在提醒后修复它需要花费几周的时间来清理它遗留的错误数据；
* 你部署了错误，然后发现你的公司45分钟后破产了。你的工作、团队、你的组织和计划，都没了。

忽略类型和运行时的概念，很明显不同的环境有不同的安全级别。只要捕捉够早或异步debug，甚至运行时错误都会造成更小的影响，Python中的习惯，当不匹配时在运行时抛出`TypeError`，似乎必Php中当不匹配时进行强制执行要安全的多。

### 什么是类型安全

**类型安全是利用我们在运行时对一个值的了解来尽量降低大部分错误的后果。**

比如，一个”较小的后果“可以被看做是开发期间就能够发现的易于理解的编译错误，然后花费30s完成修复。

这个定义直接准照上面我们对”类型“和”安全“的定义。这比很多类型安全的定义都要宽泛，特别是：

* 类型安全不是编写Haskell，这个概念要更为宽泛；
* 类型安全不是避免可变状态，除非它有助于我们的目标；
* 类型安全不是一个绝对的目标，是一个尽量和优化的属性；
* 类型安全对每个人都不同；如果不同的人犯不同的错，这些错都有不同的危害级别，他们需要完善不同的事情来尽力优化这些错误的危害；
* 如果错误消息不可理解并难于解决，编译器甚至也会有严重的影响。一个能在10s内修复的优雅的编译器消息和一个需要半小时才能理解的巨大的编译器错误消息是完全不同的。

类型安全的定义多种多样；如果你问一个C++开发者、一个Python的网站开发者或者一个研究编程语言的教授，他们会给出各自截然不同的定义。本文中，我们会使用上面宽泛的定义。

### Scalazzi Scala

很多人思考了很多关于若何以类型安全的方式编写Scala。所谓的Scala语言的“Scalazzi Subset”是其中一个哲学：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/Scalazzi.jpg" style="display:block;width:50%;" alt="NAME" align=center /> </div>

当然这些指导方针有很多地方需要讨论，我们会花一些时间浏览其中一部分，同时我发现了一些有意思的地方：

* 避免空值
* 避免异常
* 避免副作用

#### 避免空值

使用`null`来描述一些空的、未初始化或不可用的值会很吸引人，比如，一个未初始化的值：

	class Foo{
  		val name: String = null // override this with something useful if you want
	}

或者是传入到函数的一个”没有值“的参数：

	def listFiles(path: String) = ...

	listFiles("/usr/local/bin/")
	listFiles(null) // no argument, default to current working directory

“Scalazzi Scala”告诉我们要避免这样做，并给了一个很好的理由：

* `null`会出现在程序的各个角落，任何一个变量或值，没有办法控制那些变量是`null`而哪些不是；
* `null`在你的程序中到处传播：可以将`null`传入函数，赋给其他变量，或存入集合。

最终，这意味着`null`值会在原理他们初始化的地方一起错误，然后就很难被追踪。当有些地方被`NullPointerException `终止，你需要首先找到那些可疑的变量(每行代码或许会有很多变量)，然后进行追踪，比如函数的传入和传出，集合的存储和检索，直到你找到`null`的来源。

在Python这样的动态语言中，这种类型的错误值传播很普通，寻常不会贯穿整个程序来进行追踪，然后到处添加`print`语句，尝试去找到初始值的来源。通常有人简单的将参数混入到一个函数，传入一个`user_id`而不是`user_email`或其他不重要的值，但是会照成很大的后果来追踪和调试。

在一个带有类型检查器的编译型语言，比如Scala，许多这样的错误会在你运行编译器之前就能捕获：在于其为一个`String`的地方传入`Int`或`Seq[Double]`会得到一个类型错误。并不是所有的错误都会被捕捉，但是会捕捉大部分严重的错误。预期为不是`null`的地方传入一个`null`除外。

这里有一些`null`的备选方案：

**如果想要表达一个可能存在的值，一个函数参数或者一个需要被覆写的类属性：**

	class Foo{
  		val name: String = null // override this with something useful if you want
	}

考虑使用`Option[T]`替换：

	class Foo{
  		val name: Option[String] = None // override this with something useful if you want
	}

`"foo"`和`null`取而代之为`Some("foo")`和`None`看起来很相似，但是这样做的话所有人都会知道它可能为`None`，而不会像如果将一个`Some[String]`放到预期为`String`的地方然后跟`null`得到一个编译错误。

**如果使用`null`作为一个未初始化`var`值的占位符：**

	def findHaoyiEmail(items: Seq[(String, String)]) = {
  		var email: String = null // override this with something useful if you want

  		for((name, value) <- items){
			if (name == "Haoyi") email = value
  		}
  
  		if (email == null) email = "Not Found"
  		doSomething(email)
	}

考虑替换为`val`并一次完成声明和初始化：

	def findHaoyiEmail(items: Seq[(String, String)]) = {
	  val email = 
	    items.collect{case (name, value) if name == "Haoyi" => value} 
	         .headOption
	         .getOrElse("Not Found")
	  doSomething(email)
	  
如果你不能够在一行代码内初始化`email`的值，Scala支持你将片段的代码放到柯里化的`{}`中同时将其赋给一个`val`, 因此, 大部分你需要稍后初始化为`var`的代码都可以放到一个`{}`中然后声明并初始化为一个`{}`.

	def findHaoyiEmail(items: Seq[(String, String)]) = {
	  val email = {
	    ...
	  }
	  doSomething(email)
	}

这样做的话,我们就能控制`email`永远不会是一个`null`.

通过简单的在程序中避免`null`,你并没有改变理论状况, 理论上有人可以传入一个`null`,你会在同样的地方追踪那些难于调试的问题.但是你改变了实践环境: 不会花费更少的实践来追踪难于调试的`NullPointerException`问题.

#### 避免异常

异常基本上是一段代码的额外返回值.任何你写的代码都可以通过`return`关键字以正常的方式返回,或者简单的返回最后一个代码块的表达式,或者是抛出一个异常. 这些异常会包含任意的数据.

虽然一些其他语言比如Java,用编译器来检查你可以确定的能够抛出的异常,它的"受检异常"也并不是很成功: 它的不便之处在于必须要声明你抛出的需要检查的异常,以至于人们只是给他们的方法都使用一个`throws Exception`,或者捕获受检异常后重新作为未检查的运行时异常抛出.后期的语言比如C#和Scala完全抛弃了这种受检异常的思想.

为什么你不可以使用异常:

* 你没有办法静态的知道一段代码都能抛出哪些种类的异常. 即你不知道是否处理了代码所有可能的返回类型.
* 你抛出的异常的注解是可选的,and trivially fall out of sync with reality as development happens and refactorings occur.
* 他们是传播的,so even if a library you're using has gone through the discipline of annotating all its methods with the exceptions they throw, the chances are in your own code you'll get sloppy and won't.

与其返回一个异常,在只有一种失败模式的函数中,你可以返回一个`Option[T]`来表示结果,或者`Either[T, V]`,再或者是你自己定义的密闭trait来表示有多重失败模式的返回结果.
	
	sealed trait Result
	case class Success(value: String) extends Result
	case class InvalidInput(value: String, msg: String) extends Result
	case class SubprocessFailed(returncode: Int, stderr: String) extends Result
	case object TimedOut extends Result
	
	def doThing(cmd: String): Result = ???

使用密闭trait方式,你可以更易于与用户沟通存在的准确错误,在每种场景可用的数据,同时当用户对`doThing`的结果进行`match`时,如果少了一个场景,编译器则会给出一个警告.

通常,你并不能去除所有异常:

* 任何非一般的程序都很难去列出它所有可能的失败模式
* 许多都是非常罕见的,你实际上是想捕获他们的大部分然后通过一些通用的方式处理,比如: 写入日志或上报,或重试逻辑,你甚至不知道是什么引起的
* 对这些罕见的错误模式,可以吧错误信息写入日志,然后进行详细的手动检查,这也你能做的最好方式了

然而,尽管有堆栈追踪(stack trace),找出这些预期之外异常的真正原因仍然要比使用`Option[T]`在编译器就发现错误要花费的时间更多.

Scala编程中涉及的异常:

* NullPointerExceptions
* MatchError: 来自不健全的模式匹配
* IOExceptions: 来自文件系统的各种问题或网络错误
* ArithmeticException: 除0时的错误
* IndexOutOfBoundsException: 搞砸数组的时候
* IllegalArgumentException: 滥用第三方代码的时候

仍然还有更多,但是并不需要完全去管,尽量在代码中使用`Option[T], Either[T, V], sealed trait`来使编译器能有更多的机会帮你进行错误检查.

#### 避免副作用

至少在Scala中,编译器不会提供副作用的追踪.

下面是一个例子:

	var result = 0
	
	for (i <- 0 until 10){
	  result += 1
	}
	
	if (result > 10) result = result + 5 
	
	println(result) // 50
	makeUseOfResult(result)

我们将`value`初始化为一个占位值,然后利用副作用来修改`result`的值,然后为`makeUseOfResult`函数使用.

这里有很多地方会出错,你可能会意外的得到有一个突变:

	var result = 0
	
	for (i <- 0 until 10){
	  results += 1
	} 
	
	println(result) // 45
	makeUseOfResult(result) // getting invalid input!

这些可以看做是很明显的错误,但如果这个片段有1000行而不是10行,在重构中很容易出错.他以为着`makeUseOfResult`得到一个无效的输入并处理错误.这里有另一个常见的错误模式:

	var result = 0
	
	foo()
	
	for (i <- 0 until 10){
	  results += 1
	}
	
	if (result > 10) result = result + 5 
	
	println(result) // 50
	makeUseOfResult(result)
	
	...
	
	def foo() = {
	  ...
	  makeUseOfResult(result)
	  ...
	}

这里甚至在`result`被初始化之前就开始使用它了.

下面的方式可以避免副作用:

	val summed = (0 until 10).sum
	
	val result = if (summed > 10) summed + 5 else summed
	
	println(result) // 50
	makeUseOfResult(result)

#### Scalazzi Scala的局限

下面的代码完全符合上面定义的`Scalazzi Scala`,但会让人感到很乱:

	def fibonacci(n: Double, count: Double = 0, chain: String = "1 1"): Int = {
	  if (count >= n - 2) chain.takeWhile(_ != ' ').length
	  else{
	    val parts = chain.split(" ", 3)
	    fibonacci(n, count+1, parts(0) + parts(1) + " " + chain)
	  }
	}
	for(i <- 0 until 10) println(fibonacci(i))
	1
	1
	1
	2
	3
	5
	8
	13
	21
	34

这个代码是正确的,完全遵守了"Scalazzi Scala"的指导方针:

* 没有Null
* 没有异常
* 没有`isInstanceOf`或`asInstanceOf`
* 没有副作用并且所有值是不可变的
* 没有`classOf`和`getClass`
* 没有反射

但是人们会认为他是可怕的不安全的代码,原因在于下面的"Structured Data".

#### 结构化数据

并非所有的数据都有相同的"形状",如果一些数据包含`(name, phone-number)`这样的对,有多重方式可以存储他们:

* `Array[Byte]`: 这是文件系统存储他们的方式,如果你把他们存到磁盘,这就是他们的形式.
* `String`: 在编辑器中打开,会看到这样的形式.
* `Seq[(String, String)]`
* `Set[(String, String)]`
* `Map[String, String]`

这些都是有效的方式,如何来选择呢?

##### 避免字符串有利于结构化数据

有时候会将数据存为`String`,然后在使用时在使用切片取出其中的不同数据,这样做会带来意外的问题.

##### Encode Invariants in Types

##### 自描述数据

##### 避免整数枚举
	
	val UNIT_TYPE_UNKNOWN = 0
	val UNIT_TYPE_USERSPACEONUSE = 1
	val UNIT_TYPE_OBJECTBOUNDINGBOX = 2

这个代码中有一些好处:

* `Int`类型消耗廉价,需要很少的内存来存储和传递
* 避免各种数字这样的魔术代码到处都是,最终难以分辨

但是这种方式并不安全,更安全的方式会是这样:

	sealed trait UnitType  
	object UnitType{
	  case object Unknown extends UnitType
	  case object UserSpaceOnUse extends UnitType
	  case object ObjectBoundingBox extends UnitType
	}

或者:

	// You can also make it take a `name: String` param to give it a nice toString 
	case class UnitType private () 
	object UnitType{
	  val Unknown = new UnitType
	  val UserSpaceOnUse = new UnitType
	  val ObjectBoundingBox = new UnitType
	}

这两种方式都是讲`UnitType`标记为一个实际的值,而不会想仅仅一个数字一样能够修改.

##### 避免字符串标记
	
	val UNIT_TYPE_UNKNOWN = "unknown"
	val UNIT_TYPE_USERSPACEONUSE = "user-space-on-use"
	val UNIT_TYPE_OBJECTBOUNDINGBOX = "object-bounding-box"

这样做仍然不安全,可以对`UNIT_TYPE`调用任何字符串的方法,并且能够使用任何字符串替换,更好的方式是这样:

	sealed trait UnitType
	object UnitType{
	  case object Unknown extends UnitType
	  case object UserSpaceOnUse extends UnitType
	  case object ObjectBoundingBox extends UnitType
	}
	
	// Or perhaps
	
	class UnitType private ()
	object UnitType{
	  val Unknown = new UnitType
	  val UserSpaceOnUse = new UnitType
	  val ObjectBoundingBox = new UnitType
	}

#### 包装整数ID

自增的ID经常是`Int`或`Long`,UUID可能是`String`或`java.util.UUID`,与`Int`或`Long`不同的是,ID都有一个唯一属性:

* 所有的算术运算一般都没有意义
* 不同的ID不能交换:比如一个`userId: Int`和一个函数`def deploy(machineId: Int)`,`deploy(userId)`这样的调用是不希望出现的

最好的方式是使用不同的类将这些ID进行包装:

	case class UserId(id: Int)
	case class MachineId(id: Int)
	case class EventId(id: Int)
	...

或者自定义类型:

	type UID = Int

然后使用:

	val userId: UID = 2