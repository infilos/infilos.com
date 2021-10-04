---
type: docs
title: "多变量赋值"
linkTitle: "多变量赋值"
weight: 39
---

如果需要以简便的方式对多变量赋值，比如：

```scala
var MONTH = 12; var DAY = 24 
var (HOUR, MINUTE, SECOND) = (12, 0, 0)
```

但是结果并不符合预期：

```scala
MONTH: Int = 12
DAY: Int = 24
<console>:11: error: not found: value HOUR
       var (HOUR, MINUTE, SECOND) = (12, 0, 0)
            ^
<console>:11: error: not found: value MINUTE
       var (HOUR, MINUTE, SECOND) = (12, 0, 0)
                  ^
<console>:11: error: not found: value SECOND
       var (HOUR, MINUTE, SECOND) = (12, 0, 0)
                          ^
```

Scala 允许使用大写字母作为普通的变量名，就和 MONTH、DAY 一样并没有报错。但是第二条语句中使用的多变量赋值方式却不同。

因为**多变量赋值**实质上基于模式匹配，但是在模式匹配中，以大写字母开头的变量代表着特殊的意义：**它们是稳定标示符**。

稳定标识符是给常量预留的，比如：

```scala
final val TheAnswer = 42
def checkGuess(guess: Int) = guess match { 
	case TheAnswer => "Your guess is correct" 
	case _ => "Try again" 
}
scala> checkGuess(21) 	// res8: String = Try again
scala> checkGuess(42) 	// res9: String = Your guess is correct
```

相反，小写的变量名定义为**变量模式**，表示对变量的赋值：

```scala
var (hour, minute, second) = (12, 0, 0)
// hour: Int = 12 minute: Int = 0 second: Int = 0
```

因此在一开始的例子中，并不是对变量的赋值，而是对常量的匹配。

## 总结

如果想要使用大写的变量名，在极端的情况下，会对当前作用域中的值进行匹配，这个模式匹配会编译成功，并且最终的结果依赖于值是否真正匹配：

```scala
val HOUR = 12; val MINUTE, SECOND = 0;
scala> var (HOUR, MINUTE, SECOND) = (12, 0, 0)	// 1 - 匹配成功
val HOUR = 13; val MINUTE, SECOND = 0;
scala> var (HOUR, MINUTE, SECOND) = (12, 0, 0) 	// 2 - 匹配失败
scala.MatchError: (12,0,0) (of class scala.Tuple3) ...
```

在上面的第一个语句中，即便是匹配成功也不会进行任何赋值操作：**稳定标示符在模式匹配期间不会进行任何赋值**。

小写的变量名同样可以使用重音符(`)包围的方式当做稳定标示符，**同时它们必须是 val**,因此把他们当做常量来处理：

```scala
final val theAnswer = 42 
def checkGuess(guess: Int) = guess match { 
	case `theAnswer` => "Your guess is correct" 
	case _ => "Try again" 
}

```

大写的变量名并声明为 var ，在 Scala 中是不推荐的做法，而且要完全避免。使用大写变量名来声明常量，同时，常量声明为 final。这样避免被子类覆写，同时编译器将他们内联(inline)以提升性能。