---
type: docs
title: "Inject Type"
linkTitle: "Inject Type"
weight: 33
---

Scala 注解作为元数据或额外信息添加到程序源代码。类似于注释，注解可以添加到变量、方法、表达式或任何其他的程序元素。

注解可以添加到任何类型的定义或声明上，包括：`var, val, class, object, trait, def, type`。

可以同时添加多个注解，顺序无关。

给主构造器添加注解时需要将注解放在主构造器之前：

```scala
class Credentials @Inject() (var username: String, var password: String)
```

给表达式添加注解时，需要在表达式之后添加冒号，然后添加注解本身：

```scala
(myMap.get(key) : @unchecked) match {...} 
```

可以为类型参数添加注解：

```scala
class MyContainer[@specialized T]
```

只对实际类型的注解应该放在类型名之前：

```scala
String @cps[Unit] // @cps带一个类型参数 
```

声明一个注解的语法类似于：

```scala
@annot(exp_{1}, exp_{2}, ...)  {val name_{1}=const_{1}, ..., val name_{n}=const_{n}}
```

`annot`用于指定注解的类型，所有的注解都要包含这个部分。一些注解不需要提供参数，因此圆括号可以被省略或者提供一个空的圆括号。

传递给注解的精确参数需要依赖于注解类的实际定义。大多数注解执行器支持直接的常量，比如`Hi`或`678`。关键字`this`可以用于在当前作用域引用的其他变量。

类似`name=const`这样的参数可以在比较复杂的拥有可选参数的注解中见到。这个参数是可选的，并且可以按任意顺序指定。等到右边的值建议使用一个常量。

Java 注解的参数类型只能是：数值型字面量、字符串、类字面量、枚举、其他注解，或上述类型的数组但不能是嵌套数组。

Scala 注解的参数可以是任何类型。

## Scala 中的标准注解

- `scala.SerialVersionUID`：为一个可序列化的类指定一个`SerialVersionUID`字段
- `scala.deprecated`：表示这个定义已经被移除，即废弃的定义
- `scala.volatile`：告诉开发者在并发程序中允许使用可变状态
- `scala.transient`：标记为非持久字段
- `scala.throws`：指定一个方法抛出的异常
- `scala.cloneable`：标明一个类以复制(cloneable)的方式应用(apply)
- `scala.native`：原生方法的标记
- `scala.inline`：这个方法上的注解，请求编译器需要尽力内联这个被注解的方法
- `scala.remote`：标明一个类以远程(remotable)的方式应用(apply)
- `scala.serializable`：标明一个类以序列化(serializable)的方式应用(apply)
- `scala.unchecked`：适用于匹配表达式中的选择器。如果存在，表达式的警告会被禁止
- `scala.reflectBeanProperty `：当附加到一个字段时，根据`JavaBean `的管理生成 getter 和 setter 方法

## 废弃注解：@deprecated

有时需要写一个类或方法，后来又不再需要。可以为类或方法添加一个提醒一面他人使用，但是为了兼容性有不能直接移除。方法或类可以使用`@deprecated`标记，然后在使用时会有一个提醒。

## 不稳定字段：@volatile

有些开发者想要在并发程序中使用可变状态，这种场景中可以使用`@volatile`注解，通知编译器这个变量会被多个线程使用。

## 二进制序列化：@serializable/@SerialVersionUID/@transient

序列化框架将对象转换为流式字节，以节省磁盘占用或网络传输。Scala 并没有自己的序列化框架。`@serializable`注解表示一个类是否可以被序列化。默认的，类是不支持序列化的，因此需要添加该注解。

`@SerialVersionUID`用于处理可序列化的类并根据时间改变，自增数值可以以`@SerialVersionUID(678)`的方式附上当前的版本，678 即为自增 ID。

如果一个字段被标记为`@transient`，框架序列化相关的对象是不会保存该字段。当该对象被重新加载时，该字段会被设置为一个默认值。

## 自动生成 getter/setter 方法

一个带有`@scala.reflect.BeanProperty`注解的字段，编译器会自动为其生成 getter 和 setter 方法。

## 模式匹配忽略部分用例：Unchecked

`@unchecked`注解通过编译器在模式匹配时解释。告诉编译器如果匹配语句遗漏了可能的 case 时不用警告。

