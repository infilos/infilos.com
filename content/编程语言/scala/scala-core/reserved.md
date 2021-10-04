---
type: docs
title: "保留字"
linkTitle: "保留字"
weight: 31
---

## 关键字和保留字符

```scala
// 关键字符
<-			// 用于 for 表达式，从生成器(generator)中分离元素
=>			// 用于函数类型，函数字面值和引入(import)重命名

# 保留字符
( )			// 划界表达式和参数
[ ]			// 划界类型参数
{ }			// 划界块(block)
.			// 方法调用和路径分割
// /* */	// 注释
# 			// 用于类型标记
:			// 类型归属或上下文界限(context bounds)
<: >: <%	// 上界、下界或视图(view)界限
" """		// 字符串
'			// 标示符号或字符
@			// 注解和模式匹配中的变量绑定
`			// 标示常量或使称为任意标示符
,			// 参数分割
;			// 语句分割
_*			// 可变参数展开
_			// 不同场景有多种意义
```

### 下划线

```scala
import scala._			// 通配符，引入 Scala 包中所有的资源
import scala.{ Predef => _, _ }		// 排除，除了 Predef，引入其他所有
def f[M[_]]				// 高阶类型参数
def f(m: M[_])			// 存在的类型
_ + _ 					// 匿名函数参数占位符
m _						// 将方法转换为方法的值
m(_)					// 偏函数应用
_ => 5					// 丢弃的参数
case _ =>				// 通配符，匹配任何
f(xs: _*)				// 序列 xs 作为多个参数传入函数 f(ys: T*)
case Seq(xs @ _*)		// 将标识符 xs 绑定到所有匹配到的值
```

## 通用方法

一些标识符其实是一些类、特质、对象的方法。

```scala
List(1, 2) ++ List(3, 4)	// 将右边序列的元素追加到左边序列的末尾
List(1, 2).++(List(3, 4))	// 同上
1 :: List(2, 3)				// 将一个元素放到一个序列的首部
List(2, 3).::(1)			// 同上
1 +: List(2, 3) :+ 4		// +: 绑定到右边，:+ 绑定到左边
```

**以冒号(:)结尾的方法会绑定到右边，而不是左边，作为右边对象的一个方法。**

类型和对象同样也会有象征性的名字，比如：对于有两个类型参数的类型来说，名字可以写在参数之间，`Int <:< Any`和`<:<[Int, Any]`是相同的。

## 隐式转换提供的方法

Scala 代码会自动进行三个部分的引入：

```scala
// 顺序无关
import java.lang._
import scala._
import scala.Predef._
```

前两者用于类和单例对象，然而 `Predef`中定义了一些象征性的名字：

```scala
class <:<		// 一个 A <:< B 的实例，表示类型 A 是类型 B的子类型
class =:=		// 一个  A =:= B 的实例，表示类型 A 与类型 B 相同
object =:=
object <%< 		// removed in Scala 2.10
def ???			// 将一个方法为未实现
```

同时还有`::`，没有出现在文档中但是在注释中提到了。`Predef`通过隐式转换的方式激活一些方法。

## 语法糖和语法组合

```scala
class Example(arr: Array[Int] = Array.fill(5)(0)) {
  def apply(n: Int) = arr(n)
  def update(n: Int, v: Int) = arr(n) = v
  def a = arr(0); def a_=(v: Int) = arr(0) = v
  def b = arr(1); def b_=(v: Int) = arr(1) = v
  def c = arr(2); def c_=(v: Int) = arr(2) = v
  def d = arr(3); def d_=(v: Int) = arr(3) = v
  def e = arr(4); def e_=(v: Int) = arr(4) = v
  def +(v: Int) = new Example(arr map (_ + v))
  def unapply(n: Int) = if (arr.indices contains n) Some(arr(n)) else None
}
val ex = new Example
println(ex(0))  // means ex.apply(0)
ex(0) = 2       // means ex.update(0, 2)
ex.b = 3        // means ex.b_=(3)
val ex(c) = 2   // calls ex.unapply(2) and assigns result to c, if it's Some; throws MatchError if it's None
ex += 1         // means ex = ex + 1; if Example had a += method, it would be used instead
```

```scala
(_+_) // An expression, or parameter, that is an anonymous function with
      // two parameters, used exactly where the underscores appear, and
      // which calls the "+" method on the first parameter passing the
      // second parameter as argument.
```

