---
type: docs
title: "CH08-Scala APIs"
linkTitle: "CH08-Scala APIs"
weight: 8
---

与 Java 的区别在于规则的构造过程，在 Scala 中使用了特殊的 Scala DSL 构造。相比 Java API，Scala API 更具优势：

- 更加简明的规则构建 DSL(Scala 语言的丰富表现力)。
- 通过对值栈的进一步抽象隐藏了值栈，增加了类型安全性(Scala Type Inference)。
- 高阶规则构造。
- 更快的初次规则构建(不再有昂贵的解析器扩展步骤)。

## Rule Construction

一个 PEG 由任意数量的规则组成，规则又可以由其他规则、终止符、或下表中的原语规则组成：

| Name           | Common Notation | Primitive     |
| :------------- | :-------------: | :------------ |
| Sequence       |       a b       | a ~ b         |
| Ordered Choice |     a &#124; b     | a &#124; b       |
| Zero-Or-More   |       a *       | zeroOrMore(a) |
| One-Or-More    |       a +       | oneOrMore(a)  |
| Optional       |       a ?       | optional(a)   |
| And-Predicate  |       & a       | &(a)          |
| Non-Predicate  |       ! a       | !a            |

除了以上原语方法，还有以下原语可供使用：

| Method/Field        | Description                   |
| :------------------ | :---------------------------- |
| ANY                 | 匹配任何除了 EOI 的单个字符   |
| NOTHING             | 不匹配任何，总是失败          |
| EMPTY               | 不匹配任何，总是成功          |
| EOI                 | 匹配特殊的 EOI 字符           |
| ch(Char)            | 创建一个匹配单个字符的规则    |
| {String} ~ {String} | 匹配给定的字符范围            |
| anyOf(String)       | 匹配给定字符串中的任意字符    |
| ignoreCase(Char)    | 匹配单个字符且忽略大小写      |
| ignoreCase(String)  | 匹配整个字符串且胡烈大小写    |
| str(String)         | 创建一个匹配整个字符串的      |
| nTimes(Int, Rule)   | 创建一个匹配子规则 N 次的规则 |

## Parser Actions

在 Parboiled Java 中需要以布尔表达式的形式设置解析器动作，然后再被自动转换为解析器动作规则。没有进一步的动作类型来支持 Parboiled Java 对值栈操作元素数量进行区分。这意味着 Java 开发者不能依赖编译器来检测解析器动作对值栈操作的一致性(主要是元素数量)。因此在动作的设计期间需要更多对人的规范约束。

在 Parboiled Scala 中，Scala 的类型推断能力使得解析器动作支持比 Java 中更高级别的抽象。在 Scala 解析器动作中，无需对值栈进行操作，而是将其指定为函数。因此，它们不仅仅是简单的代码块，其本身就是类型。

根据规则中包含的解析器动作，规则的实际类型会发生变化。对值栈没有任何影响的规则类型为 Rule0。将类型为 A 的值对象推送到值栈的规则具有类型 `Rule1[A]`。导致类型分别为 A 和 B 的两个值对象被推送到值栈的规则类型为 `Rule2[A,B]`。导致类型为 Z 的一个值对象从堆栈中弹出的规则具有类型 `PopRule1[Z]`。目前共 15 种具体的规则类型。

这种稍微复杂的类结构允许 Scala 在规则类型中进行编码，以确定规则如何影响解析器值堆栈，并确保所有解析器操作正确地协同工作以生成解析器最终结果值。请注意，这不会对值对象的类型施加任何限制！

支持 3 种形式的解析器动作：

1. 动作操作符
2. push/test/run 方法
3. 独立动作

### Action Operators

共定义了 9 种动作操作符。每种都会链接一个动作函数到语法规则结构，但与它们的动作函数参数的类型和语义不同。下表是一个概览：

| Action Result | Action Argument(String) | Action Argument(Value Object Pop) | Action Argument(Value Object Peek) | Action Argument(Char) | Action Argument(IndexRange) |
| :------------ | :---------------------- | --------------------------------- | ---------------------------------- | --------------------- | --------------------------- |
| Value Object  | ~>                      | ~~>                               | ~~~>                               | ~:>                   | ~>>                         |
| Boolean       | ~?                      | ~~?                               | ~~~?                               |                       |                             |
| Unit          | ~%                      | ~~%                               | `~~~%`                             |                       |                             |

以单个 `~` 字符起始的操作符通常是解析器动作接收已匹配输入文本的方式。其参数是一个类型为 `String => ...` 的函数。该操作符内部会创建一个新的动作规则，在运行时，将与紧邻的规则匹配的输入文本作为参数传递给该函数。

以 `~~` 和 `~~~` 字符起始的操作符接收一个或多个值对象作为参数。

以 `>` 字符结尾的操作符创建一个或多个新的值对象，在动作函数运行之后推送到值栈。这些动作结构值的类型会被编码到操作符的返回类型。

以 `?` 字符结尾的操作符接收一个返回布尔值的函数作为语义判定。如果动作函数返回 false 则停止当前规则序列的求值，即为匹配，然后强制解析器回退并查找其他匹配可能。

以 `%` 字符结尾的操作符支持你运行任意逻辑而不会对处理过程产生影响。其动作函数返回 Unit，一旦解析器经过，它们就会被运行。

### push/test/run 方法

上述讨论的动作操作符均为将你的动作链接到当前的解析处理过程，要么是接收已匹配的输入文本作为参数，要么是生成新的栈值元素。但有时你的动作并不需要任何输入，因为其在规则结构中的位置就是其需要的所有上下文。这时你可以使用 push/test/run 方法来实现与上述讨论的操作符相同的功能，这些方法由 Parser 特质提供。

由这些方法构造的动作规则可以通过被链接在一起。如下所示：

```scala
def JsonTrue = rule { "true" ~ push(True) }
```

### 独立动作

独立动作是以 Context 对象作为参数的独立函数。它们可以像普通规则一样被使用，因为 Parser 特质提供了以下两种隐式转换：

| Method                                          | Semantics        |
| ----------------------------------------------- | ---------------- |
| toRunAction(f:(Context[Any]) => Unit):Rule0     | 通用非判断动作   |
| toTestAction(f:(Context[Any]) => Boolean):Rule0 | 通用语义判定动作 |

当前解析的 Context 为通用动作提供了对解析器的所有状态访问能力。它们可以通过 getValueStack 方法来修改解析器的值栈。但并不推荐这种用法，因为这将导致 Scala 编译器无法有效的验证值栈操作的一致性。

### “withContext” 动作

Parser 特质提供的另一个便利的工具是 withContext 方法，通过该方法，你可以包装一个动作函数然后再将其传递给动作操作符。该方法支持你的动作函数除了其常规的参数之外还能接收当前解析器的 Context。

withContext 的签名类似如下定义：

```scala
def withContext[A, B, R](f: (A, B, Context[_]) => R): ((A, B) => R)
```

因此，被该方法包装的动作函数在外部会显示为一个函数，比如，弹出值栈的两个对象并生成一个新的值。但是，在内部你的动作同样也可以接受到当前上下文的实例，比如可以查看当前输入位置以及行号。

## Parser Tesing

从 0.9.9.0 开始提供了一个 ParboiledTest 特质来简化测试的开发工作。Parboiled 使用它来完成内部测试，你可以参考 [WithContextTest](https://github.com/sirthias/parboiled/blob/master/parboiled-scala/src/test/scala/org/parboiled/scala/WithContextTest.scala) 来查看应用方式。

## Examples

- [Simple Calculator](https://github.com/sirthias/parboiled/wiki/Simple-Calculator)
- [JSON Parser](https://github.com/sirthias/parboiled/wiki/JSON-Parser)
