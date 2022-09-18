---
type: docs 
title: "基本认识"
linkTitle: "基本认识"
weight: 1
---

{{% pageinfo color="primary" %}}
[原文连接：https://www.toptal.com/dart/dartlang-guide-for-csharp-java-devs](https://www.toptal.com/dart/dartlang-guide-for-csharp-java-devs)
{{% /pageinfo %}}

Dart 编程语言之所以重要，有以下几个原因:

- 它兼备了两种语言的优点: 它是一种编译的、类型安全的语言(如 C# 和 Java)，同时也是一种脚本语言(如 Python 和 JavaScript)。
- 它转换成 JavaScript 用于 Web 前端。
- 它可以运行在任何平台上，并编译为本地移动应用，所以你几乎可以使用它做任何事情。
- Dart 在语法上类似于 C# 和 Java，所以学起来很快。

我们这些来自大型企业系统的 C# 或 Java 世界的人已经知道为什么类型安全、编译时错误和检查器很重要。我们中的许多人都在犹豫是否采用“脚本”语言，因为担心会失去我们所习惯的所有结构、速度、准确性和可调试性。

但随着 Dart 的发展，我们不需要放弃这些。我们可以用同一种语言编写移动应用程序、Web 客户端和后端——并获得我们仍然热爱 Java 和 C# 的所有东西!

为此，让我们浏览一些对 C# 或 Java 开发人员来说很新的关键 Dart 语言示例，我们将在最后的 Dart 语言 PDF 中进行总结。

注意:本文仅涉及 Dart 2.x。版本 1。x 不是“完全熟”的——特别是，类型系统是咨询型的(像 TypeScript)，而不是需要型的(像 C# 或 Java)。

## 1. 代码组织

首先，我们将讨论最重要的区别之一:如何组织和引用代码文件。

### 源文件、作用域、命名空间、引入

在 C# 中，类的集合被编译为程序集。每个类都有一个名称空间，名称空间通常反映文件系统中源代码的组织——但是最后，程序集不保留任何关于源代码文件位置的信息。
在 Java 中，源文件是包的一部分，名称空间通常符合文件系统的位置，但最终，包只是类的集合。
因此，两种语言都有一种方法使源代码在一定程度上独立于文件系统。

相比之下，在 Dart 语言中，每个源文件必须导入它引用的所有内容，包括其他源文件和第三方包。没有相同的命名空间，通常通过文件系统位置来引用文件。变量和函数可以是顶层的，而不仅仅是类。在这些方面，Dart 更像脚本。
因此，您需要将思路从“类的集合”转变为“包含的代码文件序列”。
Dart 支持包组织或不使用包的临时组织。让我们从一个没有包的例子开始，来说明包含的文件的顺序:

```dart
// file1.dart
int alice = 1; // top level variable
int barry() => 2; // top level function
var student = Charlie(); // top level variable; Charlie is declared below but that's OK
class Charlie { ... } // top level class
// alice = 2; // top level statement not allowed

// file2.dart
import 'file1.dart'; // causes all of file1 to be in scope
main() {
    print(alice); // 1
}
```

源文件中引用的所有内容都必须在该文件中声明或导入，因为没有“项目”级别，也没有其他方法在范围中包含其他源元素。
在 Dart 中，名称空间的唯一用途是为导入提供一个名称，这将影响您如何从该文件引用导入的代码。

```dart
// file2.dart
import 'file1.dart' as wonderland; 
main() {
    print(wonderland.alice); // 1
}
```

### 包：package

上面的例子在不使用包的情况下组织代码。为了使用包，代码要以更特定的方式组织起来。下面是一个名为apple的包布局示例:

- `apples/`
  - `pubspec.yaml`—定义包名、依赖，以及其他设置
  - `lib/`
    - `apples.dart`—imports and exports; 其他人通过引入该文件来消费这个包
    - `src/`
      - `seeds.dart`—其他代码
  - `bin/`
    - `runapples.dart`—包含主函数，作为入口点 (如果这是一个可运行的包或包含可运行的工具)

然后你可以导入整个包而不再是导入单个文件:

```dart
import 'package:apples';
```

重要的应用程序应该始终组织为包。这减少了在每个引用文件中重复文件系统路径的工作量;另外，它们跑得更快。它还可以很容易地在 pub.dev 上共享您的包，其他开发人员可以很容易地获取它供自己使用。应用程序使用的包会导致源代码被复制到文件系统中，所以你可以随心所欲地深入调试这些包。

## 2. 数据类型

需要注意的是，在 Dart 的类型系统中有一些主要的差异，比如空值、数字类型、集合和动态类型。

### 到处都是 Null

由于来自 C# 或 Java，我们习惯于将基本类型或值类型与引用或对象类型区分开来。实际上，值类型是在堆栈或寄存器中分配的，值的副本作为函数参数发送。引用类型被分配到堆上，只有指向对象的指针作为函数参数发送。由于值类型总是占用内存，值类型变量不能为空，而且所有值类型成员必须有初始值。
Dart 消除了这种区别，因为所有东西都是物体;所有类型最终都派生自 Object 类型。所以，这是合法的:

```dart
int i = null;
```

事实上，所有原语都隐式初始化为 null。这意味着您不能像在 C# 或 Java 中那样假定整数的默认值为零，并且您可能需要添加 null 检查。
有趣的是，即使是 Null 也是一种类型，单词 Null 指的是 Null 的实例:

```dart
print(null.runtimeType); // prints Null
```

### 数字类型并不多

与我们熟悉的 8 到 64 位的有符号和无符号整数类型不同，Dart 的主要整数类型只是 int，一个 64 位值。(对于非常大的数字，还有 BigInt。)
由于没有字节数组作为语言语法的一部分，二进制文件内容可以作为整数列表进行处理，即 `List<Int>`。
如果你认为这肯定是非常低效的，设计师已经想到了。在实践中，根据运行时使用的实际整数值，有不同的内部表示形式。运行时不会为 int 对象分配堆内存，如果它可以优化它，并在开箱模式下使用 CPU 寄存器。另外，库byte_data 提供了 UInt8List 和其他一些优化的表示。

### 集合

集合和泛型很像我们习惯使用的东西。需要注意的主要事项是没有固定大小的数组:只要在需要使用数组的地方使用 List 数据类型即可。
此外，还提供了对三种集合类型初始化的语法支持:

```dart
final a = [1, 2, 3]; // inferred type is List<int>, an array-like ordered collection
final b = {1, 2, 3}; // inferred type is Set<int>, an unordered collection
final c = {'a': 1, 'b': 2}; // inferred type is Map<string, int>, an unordered collection of name-value pairs
```

所以，在使用Java 数组、ArrayList 或 Vector 时，使用 Dart List;或 C# 数组或 List。在使用 Java/ C# HashSet 的地方使用 Set。在使用 Java HashMap 或 C# Dictionary 的地方使用 Map。

### 动态类型、静态类型

在 JavaScript、Ruby 和 Python 等动态语言中，即使成员不存在，也可以引用它们。下面是一个 JavaScript 示例:

```javascript
var person = {}; // create an empty object
person.name = 'alice'; // add a member to the object
if (person.age < 21) { // refer to a property that is not in the object
  // ...
}
```

如果你执行以上代码， `person.age` 会是 `undefined`，但确实是可以运行。

同样地，你可以在 JavaScript 中改变变量的类型:

```javascript
var a = 1; // a is a number
a = 'one'; // a is now a string
```

相比之下，在 Java 中，你不能写像上面这样的代码，因为编译器需要知道类型，它会检查所有的操作是否合法——即使你使用 var 关键字:

```java
var b = 1; // a is an int
// b = "one"; // not allowed in Java
```

Java 只允许使用静态类型编码。(您可以使用内省来执行一些动态行为，但它不是语法的直接组成部分。)JavaScript 和其他一些纯动态语言只允许使用动态类型编码。
Dart 语言允许以下两种情况:

```dart
// dart
dynamic a = 1; // a is an int - dynamic typing
a = 'one'; // a is now a string
a.foo(); // we can call a function on a dynamic object, to be resolved at run time
var b = 1; // b is an int - static typing
// b = 'one'; // not allowed in Dart
```

Dart 具有伪类型 `dynamic`，这将导致在运行时处理所有类型逻辑。调用 `a.foo()` 的尝试不会干扰静态分析器，代码会运行，但它会在运行时失败，因为没有这样的方法。
C# 最初很像 Java，后来又加入了动态支持，所以 Dart 和 C# 在这方面是差不多的。

## 4. 函数

### 函数声明语法

与 C# 或 Java 相比，Dart 中的函数语法更轻松、更有趣。语法如下:

```dart
// functions as declarations
return-type name (parameters) {body}
return-type name (parameters) => expression;

// function expressions (assignable to variables, etc.)
(parameters) {body}
(parameters) => expression
```

比如：

```dart
void printFoo() { print('foo'); };
String embellish(String s) => s.toUpperCase() + '!!';

var printFoo = () { print('foo'); };
var embellish = (String s) => s.toUpperCase() + '!!';
```

### 参数传递

因为所有东西都是对象，包括基本类型 int 和 String，所以参数传递可能会让人困惑。虽然没有像 C# 那样传递 ref 形参，但所有的参数都是通过引用传递的，函数不能更改调用者的引用。因为对象在传递给函数时不会被克隆，所以函数可能会改变对象的属性。然而，像 int 和 String 这样的基本类型的区别实际上是没有意义的，因为这些类型是不可变的。

```dart
var id = 1;
var name = 'alice';
var client = Client();

void foo(int id, String name, Client client) {
	id = 2; // local var points to different int instance
	name = 'bob'; // local var points to different String instance
	client.State = 'AK'; // property of caller's object is changed
}

foo(id, name, client);
// id == 1, name == 'alice', client.State == 'AK'
```

### 可选参数

如果你是在 C# 或 Java 的世界里，你可能会诅咒这些令人困惑的重载方法的情况:

```java
// java
void foo(string arg1) {...}
void foo(int arg1, string arg2) {...}
void foo(string arg1, Client arg2) {...}
// call site:
foo(clientId, input3); // confusing! too easy to misread which overload it is calling
```

对于 C# 可选参数，还有另一种困惑:

```cs
// C#
void Foo(string arg1, int arg2 = 0) {...}
void Foo(string arg1, int arg3 = 0, int arg2 = 0) {...}
 
// call site:
Foo("alice", 7); // legal but confusing! too easy to misread which overload it is calling and which parameter binds to argument 7
Foo("alice", arg2: 9); // better
```

C# 不需要在调用点命名可选参数，所以用可选参数重构方法可能会很危险。如果某些调用站点在重构后恰好是合法的，编译器将不会捕获它们。
Dart 有一种更安全、更灵活的方式。首先，重载方法不受支持。相反，有两种方法来处理可选参数:

```dart
// positional optional parameters
void foo(string arg1, [int arg2 = 0, int arg3 = 0]) {...}

// call site for positional optional parameters
foo('alice'); // legal
foo('alice', 12); // legal
foo('alice', 12, 13); // legal

// named optional parameters
void bar(string arg1, {int arg2 = 0, int arg3 = 0}) {...}
bar('alice'); // legal
bar('alice', arg3: 12); // legal
bar('alice', arg3: 12, arg2: 13); // legal; sequence can vary and names are required
```

不能在同一个函数声明中使用这两种样式。

### `async` 关键字位置

C# 的 async 关键字有一个令人困惑的位置:

```cs
Task<int> Foo() {...}
async Task<int> Foo() {...}
```

这意味着函数签名是异步的，但实际上只有函数实现是异步的。上面的任何一个签名都是这个接口的有效实现:

```cs
interface ICanFoo {
    Task<int> Foo();
}
```

在 Dart 语言中，async 位于更符合逻辑的位置，表示实现是异步的:

```dart
Future<int> foo() async {...} 
```

### 作用域与闭包

像 C# 和 Java 一样，Dart 在词法上是有作用域的。这意味着在块中声明的变量在块的末尾超出了作用域。所以 Dart 处理闭包的方式是一样的。

### 属性语法

Java 普及了属性 get/set 模式，但语言中并没有针对它的任何特殊语法:

```java
// java
private String clientName;
public String getClientName() { return clientName; }
public void setClientName(String value}{ clientName = value; }
```

C# 有它的语法:

```cs
// C#
private string clientName;
public string ClientName {
    get { return clientName; }
    set { clientName = value; }
}
```

Dart 的语法支持属性略有不同:

```dart
// dart
string _clientName;
string get ClientName => _clientName;
string set ClientName(string s) { _clientName = s; }
```

## 5. 构造器

Dart 构造函数比 C# 或 Java 具有更多的灵活性。一个很好的特性是能够在同一个类中命名不同的构造函数:

```dart
class Point {
    Point(double x, double y) {...} // default ctor
    Point.asPolar(double angle, double r) {...} // named ctor
}
```

你可以只使用类名来调用默认构造函数:
在调用构造函数体之前初始化实例成员有两种简写方式:

```dart
class Client {
    String _code;
    String _name;
    Client(String this._name) // "this" shorthand for assigning parameter to instance member
        : _code = _name.toUpper() { // special out-of-body place for initializing
        // body
    }
}
```

构造函数可以运行超类构造函数并重定向到同一类中的其他构造函数:

```dart
Foo.constructor1(int x) : this(x); // redirect to the default ctor in same class; no body allowed
Foo.constructor2(int x) : super.plain(x) {...} // call base class named ctor, then run this body
Foo.constructor3(int x) : _b = x + 1 : super.plain(x) {...} // initialize _b, then call base class ctor, then run this body
```

在 Java 和 C# 中，在同一个类中调用其他构造函数的构造函数，当它们都有实现时，可能会令人混淆。在 Dart 中，重定向构造函数不能有主体，这一限制迫使程序员将构造函数层变得更清晰。
还有一个 factory 关键字允许函数像构造函数一样使用，但实现只是一个常规函数。你可以使用它来返回一个缓存实例或一个派生类型的实例:

```dart
class Shape {
    factory Shape(int nsides) {
        if (nsides == 4) return Square();
        // etc.
    }
} 

var s = Shape(4); 
```

## 6. 修饰符

在 Java 和 C# 中，我们有 private、protected 和 public 等访问修饰符。在 Dart 中，这被大大简化了:如果成员名以下划线开头，它在包内的任何地方都是可见的(包括从其他类)，而对外部调用者是隐藏的;否则，从任何地方都可以看到它。没有像 private 这样的关键字来表示可见性。
另一种修饰符控制可变性:关键字 final 和 const 就是为了这个目的，但它们的含义不同:

```dart
var a = 1; // a is variable, and can be reassigned later
final b = a + 1; // b is a runtime constant, and can only be assigned once
const c = 3; // c is a compile-time constant
// const d = a + 2; // not allowed because a+2 cannot be resolved at compile time
```

## 7. 类继承

Dart 语言支持接口、类和一种多继承。但是，没有界面关键字;相反，所有的类也是接口，所以你可以定义一个抽象类，然后实现它:

```dart
abstract class HasDesk {
    bool isDeskMessy(); // no implementation here
}
class Employee implements HasDesk {
    bool isDeskMessy() { ...} // must be implemented here
}
```

使用 extends 关键字对主沿袭进行多重继承，其他类使用 with 关键字:

```dart
class Employee extends Person with Salaried implements HasDesk {...}
```

在这个声明中，Employee 类派生自 Person 和 Salaried，但是 Person 是主要的超类而 Salaried 是 mixin(次级超类)。

## 8. 操作符

有一些有趣和有用的Dart操作符是我们不习惯的。
Cascades 允许你在任何东西上使用链接模式:

```dart
emp ..name = 'Alice' ..supervisor = 'Zoltron' ..hire();
```

spread 操作符允许将集合视为初始化器中元素的列表:

```dart
var smallList = [1, 2];
var bigList = [0, ...smallList, 3, 4]; // [0, 1, 2, 3, 4]
```

## 9. 线程

Dart 没有线程，这使得它可以转换为 JavaScript。相反，它有“隔离”，从不能共享内存的意义上讲，它们更像是独立的进程。由于多线程编程非常容易出错，因此这种安全性被视为 Dart 的优点之一。要[在隔离之间进行通信](https://renato.athaydes.com/posts/interesting-dart-features.html#isolates)，您需要在它们之间流数据;接收到的对象被复制到接收隔离的内存空间中。

## 使用 Dart 编程

如果您是一名 C# 或 Java 开发人员，您已经知道的知识将帮助您快速学习 Dart 语言，因为它被设计为熟悉的语言。为此，我们整理了一份 Dart 小抄 PDF 供你参考，特别强调了它与 C# 和 Java 等价物的重要区别:

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220909234605.png" style="display:block;margin-left:auto;margin-right:auto;width:100%;" alt="20220909234605" /></div>
