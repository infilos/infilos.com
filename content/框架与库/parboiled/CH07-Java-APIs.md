---
type: docs
title: "CH07-Java APIs"
linkTitle: "CH07-Java APIs"
weight: 7
---

Parboiled Java 的应用步骤：

1. 安装依赖。
2. 确定要解析器值栈中要参数化的类型，继承 BaseParser 实现自定义解析类。
3. 在该解析类中添加返回类型为 Rule 的规则方法。
4. 通过 `Parboiled.createParser` 创建解析器实例。
5. 调用解析器的根规则方法来创建规则树。
6. 选择 ParseRunner 的特定实现，调用其 run 方法并传入根规则和输入文本。
7. 查看 ParsingResult 对象的属性。

## Rule Construction

一个 PEG 由任意数量的规则组成，规则又可以由其他规则、终止符、或下表中的原语规则组成：

| Name           | Common Notation | Primitive      |
| :------------- | :-------------: | :------------- |
| Sequence       |       a b       | Sequence(a, b) |
| Ordered Choice |      a &#124; b       | FisrtOf(a, b)  |
| Zero-Or-More   |       a *       | ZeroOrMore(a)  |
| One-Or-More    |       a +       | OneOrMore(a)   |
| Optional       |       a ?       | Optional(a)    |
| And-Predicate  |       & a       | Test(a)        |
| Non-Predicate  |       ! a       | TestNot(a)     |

这些原语实际是 BaseParser 类的实例方法，即所有自定义解析器的必要父类。这些原语规则创建方法可以接收一个或多个 Object 参数，这些参数的类型可以是：

- 一个 Rule 实例
- 一个字符字面量
- 一个字符串字面量
- 一个字符数组
- 一个动作表达式
- 实现了 Action 接口的类的实例

除了以上原语方法，还有以下原语可供使用：

| Method/Field          | Description                        |
| :-------------------- | :--------------------------------- |
| ANY                   | 匹配任何除了 EOI 的单个字符        |
| NOTHING               | 不匹配任何，总是失败               |
| EMPTY                 | 不匹配任何，总是成功               |
| EOI                   | 匹配特殊的 EOI 字符                |
| Ch(char)              | 创建一个匹配单个字符的规则         |
| CharRange(char, char) | 匹配给定的字符范围                 |
| AnyOf(string)         | 匹配给定字符串中的任意字符         |
| NoneOf(string)        | 不匹配给定字符串中的任意字符和 EOI |
| IgnoreCase(char)      | 匹配单个字符且忽略大小写           |
| IgnoreCase(String)    | 匹配整个字符串且胡烈大小写         |
| String(string)        | 创建一个匹配整个字符串的           |

## Parser Action Expressions

Parboiled Java 的解析器可以在规则定义的任意位置包含解析器动作。这些动作可以分为 4 类。

### “Regular” objects implementing the Action interface

如果你的动作代码较多，则可以将其封装到一个实现了 Action 接口的自定义类中，然后再在规则定义方法中使用该自定义类的实例。

```java
class MyParser extends BaseParser<Object> {
    Action myAction = new MyActionClass();

    Rule MyRule() {
        return Sequence(
            ...,
            myAction,
            ...
        );
    }
}
```

如果使用这种方式，你的自定义动作类也可以实现 SkippableAction 接口以告诉解析器引擎在执行内部的语法判定时是否跳过这些动作。

### Anonymous inner classes implementing the Action interface

更多时候动作仅会包含少量的代码，这时可以直接使用匿名类：

```java
class MyParser extends BaseParser<Object> {

    Rule MyRule() {
        return Sequence(
            ...,
            new Action() {
                public boolean run(Context context) {
                    ...; // arbitrary action code
                    return true; // could also return false to stop matching the Sequence and continue looking for other matching alternatives
                }
            },
            ...
        );
    }
}
```

### Explicit action expressions

虽然匿名类要比独立的动作类定义简单一点，但仍然显得冗余。可以继续简化为一个布尔表达式：

```java
class MyParser extends BaseParser<Object> {

    Rule MyRule() {
        return Sequence(
            ...,
            ACTION(...), // the argument being the boolean expression to wrap
            ...
        );
    }
}
```

`BaseParser.ACTION` 是一个特殊的标记方法，其告诉 Parboiled 将参数表达式封装到一个单独的、自动创建的动作类中，类似上面匿名类的例子。这样的动作表达式中可以包含对本地变量的访问代码或方法参数、读写非私有的解析器字段、调用非私有的解析器方法。

此外，如果动作表达式中调用实现了 ContextAware 接口的类对象方法，将自动在调用方法之前插入 setContext 方法。比如你想将所有的动作代码移出到解析器类之外以简化实现：

```java
class MyParser extends BaseParser<Object> {
    MyActions actions = new MyActions();

    Rule MyRule() {
        return Sequence(
            ...,
            ACTION(actions.someAction()),
            ...
        );
    }
}
```

如果 MyActions 实现了 ContextAware 接口，Parboiled 将会自动在内部转换为类似下列清单的代码：

```java
class MyParser extends BaseParser<Object> {
    MyActions actions = new MyActions();

    Rule MyRule() {
        return Sequence(
            ...,
            new Action() {
                public boolean run(Context context) {
                    actions.setContext(context);
                    return actions.someAction();
                }
            },
            ...
        );
    }
}
```

注意 BaseParser 已经继承了 BaseActions，其实现了 ContextAware 接口，所以解析器类中的所有动作方法可以通过 getContext 方法获得当前的上下文。

### Implicit action expressions

大多数情况下，Parboiled 可以自动识别你的规则定义中哪些是动作表达式。比如下面的规则定义中包含了一个隐式的动作表达式：

```java
Rule Number() {
    return Sequence(
        OneOrMore(CharRange('0', '9')),
        Math.random() < 0.5 ? extractIntegerValue(match()) : someObj.doSomething()
    );
}
```

Parboiled 的检测逻辑如下：

BaseParser 中所有的默认规则创建器方法都拥有通用的 Java Object 参数，Java 编译器会自动将原始布尔表达式的结果作为一个 Boolean 对象传递。这是通过在布尔动作表达式的代码之后隐式地插入对 Boolean.valueOf 的调用来实现的。Parboiled 会在你的规则方法字节码中查找这些调用然后将其当做隐式动作表达式来处理，如果其结果直接被用作规则创建方法的参数的话。也可以通过 `@ExplicitActionsOnly` 注解(定义在解析类或规则方法上)来关闭该功能。

### Return Values

动作表达式均为布尔表达式。其返回值将影响对当前值的解析进度。如果动作表达式的结果为 false，解析将继续，就像替换动作表达式的假设解析规则失败一样。因此，你可以将动作表达式视作可以(匹配)成功或(匹配)失败的“规则”，具体则取决于其返回值。

## Value Stack

在任何特定的解析项目中，解析器动作都希望能够以某种方式来创建对应输入文本结构的自定义对象。Parboiled Java 提供了两种工具来在解析器规则中管理创建的自定义对象：

- 值栈
- 动作变量

值栈是一个简单的栈结构，作为一个临时存储为你的自定义对象提供服务。你的解析器动作可以将对象推到栈上、推出栈、推出再推入栈交换对象，等等。值栈的实现隐藏在 ValueStack 接口下面，其定义了操作值栈的一系列方法。

所有的解析器动作可以通过当前 context 的 getValueStack 来获得当前值栈。为了简化值栈操作的冗余，BaseActions 类(BaseParser)的父类提供了一些值栈操作的快捷方法，可以直接在解析器动作表达式中内联使用。

在解析器规则中使用值栈的方式通常有以下几种：

- 匹配分隔符、空格或其他辅助结构的规则通常不会影响值栈。
- 较底层的规则会从匹配到的输入中创建基本对象并推到栈上。
- 调用一个或多个底层规则的高级别规则，会从栈上推出值对象，然后创建高级别的对象并重新推到栈上。
- 根规则作为最高级别的规则会创建自定义结构的根对象。

大多时候，但一个规则被完整处理过后，最多会推一个对象到栈上(尽管在处理过程中会推多个对象到栈上)。那么你可以认为：如果规则匹配，一个规则会在栈上创建一个特定类型的对象，否则则不会影响栈。

### 规则定义须知

一条重要的原则是一个规则总是应该确保其对值栈的操作是“稳定的行为”，而无论输入是什么。因此，如果一个规则将一个特定类型的值对象推到栈上，则其应该为所有可能的输入都推一个值到栈上。如果不然，那么引用该规则之外的规则时将无法在规则匹配之后会值栈的状态进行假设，这将使动作设计复杂化。以下讨论着眼于各种 PEG 原语以及在使用影响值栈的解析器操作时需要注意的事项。

#### Sequence 规则

由于它们不提供任何可选组件，因此关于值栈操作，序列规则相当直接。它们的最终结果本质上是稳定的，仅包括所有子操作的串联。

#### FirstOf 规则

FirstOf 规则提供了几种替代子规则匹配。为了向外部提供稳定的“输出”，重要的是所有替代方案都表现出兼容的值堆栈行为。考虑以下例子：

```java
Rule R() {
    return FirstOf(A(), B(), C());
}
```

如果子规则 A 将推一个对象到栈，则 B 和 C 也需要这样做。

#### Optional 规则

Optional 规则的子规则通常不应该在值栈中添加或删除对象。由于 Optional 规则始终会匹配成功，即使其子规则不匹配，对值栈上的对象数量的任何影响都将违反“稳定行为”的原则。但是，Optional 规则可以很好的转换值栈上的内容，而避免不稳定的行为。

```java
Rule R() {
    return Sequence(
        Number(), // number adds an Integer object to the stack
        Optional(
            '+',
            Number(), // another Integer object on the stack
            push(pop() + pop()) // pop two and repush one Integer
        )
    );
}
```

该规则的行为始终是稳定的，因为它总是会将一个值推送到栈上。

#### ZeroOrMore/OneOrMore 规则

与 Optional 规则类似，不能添加或删除值栈的元素，而可以修改值栈的元素内容。

## Action Variables

对值栈的操作需要一些设计素养，同时为了类型安全，值栈中仅能使用一个较为宽泛的通用类型，然后再在解析器动作使用使用强制类型转换，这会带来维护成本。为了提供更多的灵活性，提供了动作变量功能。

通常，一个规则方法会在规则的子结构中包含多个动作表达式，以协同的方式来构造出最终规则。在很多情况下如果能够通过一个通用的临时辅助变量来访问规则中所有的动作，则会大有帮助。考虑如下例子：

```java
Rule Verbatim() {
    StringVar text = new StringVar("");
    StringVar temp = new StringVar("");
        return Sequence(
            OneOrMore(
                ZeroOrMore(BlankLine(), temp.append("\n")),
                NonblankIndentedLine(), text.append(temp.getAndSet(""), pop().getText())
            ),
            push(new VerbatimNode(text.get()))
        );
}
```

该规则用于解析 Markdown 的逐行结构，其中包含一行或多行的缩进文本。这些缩进行可以通过完全的空行来拆分，如果其跟随的有最少一个缩进行则也可以被匹配。该规则的工作是创建一个 AST 节点并初始化为匹配到的文本(不带有行缩进)。

为了能够构建该 AST 节点的文本参数，如果能够访问一个字符串变量——作为构建字符串的临时容器，则会非常有帮助。在通常的 Java 方法中可以使用一个本地变量，然而，因为规则方法仅包含规则的构造代码而非规则实际运行的代码，因此本地变量起不了作用。因为本地变量仅能在规则的构造期间而非运行期间可见。

这就是为什么 Parboiled Java 提供了一个名为 Var 的类，它可以用作规则执行阶段的本地变量。Var 对象包装一个任意类型的值，可以拥有初始值，支持对值的读写，可以在嵌套规则方法之间传递。每轮规则调用(如规则匹配重试)都会接受到自己的 Var 域，因此递归规则中的动作也会像预期一样运行。此外，Var 类还定义了一系列简便的辅助方法来简化其在动作表达式中的应用过程。

如下所示：

```java
Rule A() {
    Var<Integer> i = new Var<Integer>();
    return Sequence(
        ...,
        i.set(42),
        B(i),
        action(i.get())
    );
}

Rule B(Var<Integer> i) {
    return Sequence(
        ...,
        i.set(26)
    );
}
```

规则方法 A 传递一个其域内定义的 Var 作为参数到规则方法 B，规则方法 B 内的动作向该 Var 写入一个新值，规则方法 A 中所有运行在 B 之后的动作都能看到该新写入的 Var 的值。上面的例子中，A 中的 action 读取 Var 值时会得到 26。

## Parser Extension

当你首次调用 `Parboiled.createParser` 来构造你的解析器实例时，Parboiled Java 会在内部运行解析器扩展逻辑来为你的解析器类增加所有可能的特殊功能。因为你定义的解析器类一定不是 private 和 final 的，因此可以被子类化。新创建的类与你原有的解析器类处于同一个包下，使用原有的类名并加上 `$$parboiled` 后缀。

自动穿件的解析器子类会覆写所有返回 Rule 实例的方法。这些覆写会在某个点将调用为派给父方法(即原始解析器类中的方法)，或者甚至完全重写而不会父类方法做任何调用。

以下规则方法扩展需要完全重写而不会对父方法执行委派调用：

- 解析器动作表达式
- 动作变量

以下规则方法扩展可以无需方法重写而应用，如果方法中没有上面列出的转换时也可以调用父方法：

- @Label
- @Cache
- @SuppressNode
- @SuppressSubnodes
- @SkipNode
- @MemoMismatches

通常你不必担心是否需要进行方法覆写的问题。然而在调试环节，当你需要在规则方法中添加断点以追踪执行过程时，如果你的规则方法被重写，则端点就无法没命中。因此，比如你需要调试一个带有隐式或显式动作表达式的规则方法时，需要临时将动作表达式改写为显式匿名内部 Action 类，来避免对该规则方法的完全重写。

解析器扩展逻辑不会触碰那些不返回 Rule 实例的方法则，而是直接保留。

## Examples

- [ABC Grammar](https://github.com/sirthias/parboiled/tree/master/examples-java/src/main/java/org/parboiled/examples/abc)
- [Calculators](https://github.com/sirthias/parboiled/wiki/Calculators)
- [Java Parser](https://github.com/sirthias/parboiled/wiki/Java-Parser)
- [Time Parser](https://github.com/sirthias/parboiled/wiki/Time-Parser)

