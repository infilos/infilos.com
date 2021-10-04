---
type: docs
title: "构造器"
linkTitle: "构造器"
weight: 40
---

##  继承中的构造器初始化顺序

很多编程语言通过构造器参数类初始化类的成员变量：

```scala
class MyClass(param1, param2, ...) { 
	val member1 = param1 
	val member2 = param2 
	... 
}
```

在 Scala 中，构造器参数既是成员变量，避免了重复赋值：

```scala
class MyClass(val member1, val member2, ...) { 
	... 
}
```

但是下面的代码：

```scala
trait A { 
	val audience: String 
	println("Hello " + audience) 
}

// 通过成员实现接口字段
class BMember(a: String = "World") extends A { 
	val audience = a 						
	println("I repeat: Hello " + audience) 
}

// 通过构造器实现接口字段
class BConstructor(val audience: String = "World") extends A {
	println("I repeat: Hello " + audience)
}
new BMember("Readers") 
new BConstructor("Readers")
```

其执行结果为：

```scala
scala> new BMember("Readers") 
Hello null 					// <======= null
I repeat: Hello Readers 
res3: BMember = BMember@1aa6f6eb

scala> new BConstructor("Readers") 
Hello Readers 
I repeat: Hello Readers 
res4: BConstructor = BConstructor@64b6603a
```

这表示，A 中 audience 的值，随着该成员是在构造器参数列表中声明或在构造器体重声明而不同。

要理解这两种成员声明方式的不同，需要了解类测初始化顺序。B 的两种构造器都是以下面的形式声明：

```scala
class c(param1) extends superclass { statements }
```

`new BMember("Readers")`和`new BConstructor("Readers")`的初始化会以下面的序列进行：

1. 参数值 ”Readers“ 被求值，当然这里他直接是一个字符串，不需要计算，但如果他是一个表达式，比如`"readers".capitalize`，则会首先计算
2. 被构造的类以下面的模板进行计算：`superclass { statements }`
   1. 首先，是 A 的构造器，A 的构造体
   2. 然后，是子类构造体

因此，在 BMember 中，第一步是将 "Readers" 赋值给构造器参数 a，然后是 A 的构造器被调用，但是这时成员 audience 还没有被初始化，所以默认值为 null。接着，子类 BMember 的构造体被执行，变量 a 的值被赋值给成员 audience，最终打印出了 audience 的值。

而 BConstructor 中，”Readers“ 被计算并以直接的方式赋值给 audience，因为这是构造器参数计算的一部分。因此当 A 的构造器被调用时，audience 的值已经被初始化为 ”Readers“。

## 总结

通常，BConstructor 的模式作为首选的方式。

同样可以使用**字段提前定义(early field definition)**来实现同样的结果。这样可以支持你在构造器参数上执行额外的计算，或者以正确初始化的值来创建匿名类：

```scala
class BEarlyDef(a: String = "World") 
  extends { val audience = a } 			// 字段提前定义部分
  with A { println("I repeat: Hello " + audience) }

scala> new BEarlyDef("Readers") 
Hello Readers 
I repeat: Hello Readers 
res7: BEarlyDef = BEarlyDef@44c93da7
```

```scala
scala> new { val audience = "Readers" } with A {
	println("I repeat: Hello " + audience) } 
Hello Readers 
I repeat: Hello Readers 
res0: A = anon1@71e16512
```

**提前定义(Early definitions)**在超类构造器调用之前定义并赋值成员。

因此，加上之前顺序后的完整顺序：

1. 执行子类构造器参数求值
2. 执行字段提前定义
3. 执行父类、父特质构造器、构造体，被混入的特质按照出现的顺序从左到右执行
4. 执行子类、子特质构造体

一个完整的示例：

```scala
trait A { 
	val audience: String 
	println("Hello " + audience) 
}

trait AfterA { 
	val introduction: String 
	println(introduction) 
}

class BEvery(val audience: String) extends { 
	val introduction = { println("Evaluating early def"); "Are you there?" } } 		with A 
	with AfterA {
		println("I repeat: Hello " + audience) 
	}

scala> new BEvery({ println("Evaluating param"); "Readers" }) 
Evaluating param 				// 参数计算
Evaluating early def 			// 提前定义计算
Hello Readers 					// 第一个父类构造器、构造体计算
Are you there? 					// 第二个父类构造器、构造体计算
I repeat: Hello Readers 		// 第三个匿名父类构造器、构造体计算
res3: BEvery = BEvery@6bcc2569
```

