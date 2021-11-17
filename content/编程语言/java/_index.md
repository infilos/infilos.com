
---
type: docs
title: "Java 编程"
linkTitle: "Java 编程"
weight: 1
---

## Java 9

### 接口私有方法

支持在接口内声明私有默认方法。

### 匿名内部类类型推断

```java
List<Integer> numbers = new ArrayList<>() {
    // ..
}
```

### try-with-resources 新语法

```java
BufferedReader br1 = new BufferedReader(...);
BufferedReader br2 = new BufferedReader(...);
try (br1; br2) {
    System.out.println(br1.readLine() + br2.readLine());
}
```

### 弃用下划线标识符

```java
int _ = 10; // Compile error
```

### 警告提升

私有方法支持 @SafeVarargs。

引入废弃类型时不再警告。

## Java 11

### 本地变量类型推断

省略类型声明并不表示动态类型，而是更加智能的类型推断：

```java
var greetingMessage = "Hello!";

var date = LocalDate.parse("2019-08-13");
var dayOfWeek = date.getDayOfWeek();
var dayOfMonth = date.getDayOfMonth();

Map<String, String> myMap = new HashMap<String, String>(); // Pre Java 7
Map<String, String> myMap = new HashMap<>(); // Using Diamond operator
```

## Java 14

### Switch 表达式

新的 switch 语句：

```java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY                -> 7;
    default      -> {
        String s = day.toString();
        int result = s.length();
        yield result;
    }
};
```

可以只用作为表达式使用：

```java
int k = 3;
System.out.println(
    switch (k) {
        case  1 -> "one";
        case  2 -> "two";
        default -> "many";
    }
);
```

每个 case 都拥有自己的域：

```java
String s = switch (k) {
    case  1 -> {
        String temp = "one";
        yield temp;
    }
    case  2 -> {
        String temp = "two";
        yield temp;
    }
    default -> "many";
}
```

switch 的 case 必须详尽，这意味着 String、原始类型及其包装类型的 default 必须提供：

```java
int k = 3;
String s = switch (k) {
    case  1 -> "one";
    case  2 -> "two";
    default -> "many";
}
```

对于枚举来说，要么匹配所有子类，要么提供 default case：

```java
enum Day {
   MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

Day day = Day.TUESDAY;
switch (day) {
    case  MONDAY -> ":(";
    case  TUESDAY, WEDNESDAY, THURSDAY -> ":|";
    case  FRIDAY -> ":)";
    case  SATURDAY, SUNDAY -> ":D";
}
```

## Java 15

### 文本块

```java
String html = """
          <html>
            <body>
              <p>Hello, world</p>
            </body>
          </html>
          """;

System.out.println(html);
```

每个换行尾部都会自动追加 `\n` 标识，如果在尾部显式添加一个 `\` 字符，则表示不换行，但又增加了可读性：

```java
String singleLine = """
          Hello \
          World
          """;
```

额外缩进会被自动移除：

```java
String indentedByToSpaces = """
         First line 
         Second line
       """;

String indentedByToSpaces = """
                              First line 
                              Second line
                            """;
```

目前还不支持插值语法，但可以使用 formatted 方法：

```java
var greeting = """
    hello
    %s
    """.formatted("world");
```

### NPE 更有意义

原有异常栈：

```
node.getElementsByTagName("name").item(0).getChildNodes().item(0).getNodeValue();

Exception in thread "main" java.lang.NullPointerException
        at Unlucky.method(Unlucky.java:83)
```

新的异常栈：

```
Exception in thread "main" java.lang.NullPointerException:
  Cannot invoke "org.w3c.dom.Node.getChildNodes()" because
  the return value of "org.w3c.dom.NodeList.item(int)" is null
        at Unlucky.method(Unlucky.java:83)
```

## Java 16

### Record 类

> 与 Scala 中 case class 类似。

用于声明一个不可变的数据类。

```java
public record Point(int x, int y) { }

var point = new Point(1, 2);
point.x(); // returns 1
point.y(); // returns 2
```

该声明的含义如下:

- two `private` `final` fields, `int x` and `int y`
- a constructor that takes `x` and `y` as a parameter
- `x()` and `y()` methods that act as getters for the fields
- `hashCode`, `equals` and `toString`, each taking `x` and `y` into account

其限制如下：

- 不能含有任何非 final 字段
- 默认构造器需要包含所有字段，可以声明额外构造器以提供默认字段值
- 不能继承其他类
- 不能声明 native 方法
- 隐式 final，不能声明为 abstract

提供隐私无参构造器，也可以显式声明无参构造器以实现参数校验：

```java
public record Point(int x, int y) {
  public Point {
    if (x < 0) {
      throw new IllegalArgumentException("x can't be negative");
    }
    if (y < 0) {
      y = 0;
    }
  }
}
```

声明额外构造器必须委托给其他构造器：

```java
public record Point(int x, int y) {
  public Point(int x) {
    this(x, 0);
  }
}
```

访问器可以被覆写，其他隐私的方法如 hashCode、equals、toString 也可以被覆写：

```java
public record Point(int x, int y) {
  @Override
  public int x() {
    return x;
  }
}
```

能够声明静态或实例方法：

```java
public record Point(int x, int y) {
  static Point zero() {
    return new Point(0, 0);
  }
  
  boolean isZero() {
    return x == 0 && y == 0;
  }
}
```

可以实现 Serializable 接口，且不需要提供 serialVersionUID：

```java
public record Point(int x, int y) implements Serializable { }

public static void recordSerializationExample() throws Exception {
  Point point = new Point(1, 2);

  // Serialize
  var oos = new ObjectOutputStream(new FileOutputStream("tmp"));
  oos.writeObject(point);

  // Deserialize
  var ois = new ObjectInputStream(new FileInputStream("tmp"));
  Point deserialized = (Point) ois.readObject();
}
```

可以直接在方法体内声明 Record 类：

```java
public List<Product> findProductsWithMostSaving(List<Product> products) {
  record ProductWithSaving(Product product, double savingInEur) {}

  products.stream()
    .map(p -> new ProductWithSaving(p, p.basePriceInEur * p.discountPercentage))
    .sorted((p1, p2) -> Double.compare(p2.savingInEur, p1.savingInEur))
    .map(ProductWithSaving::product)
    .limit(5)
    .collect(Collectors.toList());
}
```

### 模式匹配：instanceof

instanceof 语法支持自动 cast：

```java
if (obj instanceof String s) {
    // use s
}
```

新的 instanceof 检查与原来的逻辑类似，但如果检查总是通过，将会直接抛出错误：

```java
// "old" instanceof, without pattern variable:
// compiles with a condition that is always true
Integer i = 1;
if (i instanceof Object) { ... } // works

// "new" instanceof, with the pattern variable:
// yields a compile error in this case
if (i instanceof Object o) { ... } // error
```

模式检查通过则提取出一个模式变量，该变量为常规的 non-final 变量类似：

- 可以被修改
- 覆盖字段声明
- 如果有相同名称的本地变量，则编译失败

模式变量可以直接用于后置的检查逻辑：

```java
if (obj instanceof String s && s.length() > 5) {
  // use s
}
```

模式变量的作用域也不仅限于检查内部：

```java
private static int getLength(Object obj) {
  if (!(obj instanceof String s)) {
    throw new IllegalArgumentException();
  }

  // s is in scope - if the instanceof does not match
  //      the execution will not reach this statement
  return s.length();
}
```

## Java 17

### Sealed 类

> 与 Scala 中 sealed trait 类似。

用于声明一个边界清晰的抽象层级。Sealed 类的子类可以选择 3 种修饰符，以约束抽象边界：

- final：子类无法再被继承
- sealed：子类仅能被允许的类继承
- non-sealed：子类可以被自由继承

```java
public sealed class Shape {
  public final class Circle extends Shape {}

  public sealed class Quadrilateral extends Shape {
    public final class Rectangle extends Quadrilateral {}
    public final class Parallelogram extends Quadrilateral {}
  }

  public non-sealed class WeirdShape extends Shape {}
}
```

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20211026220510.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

#### 替代 Enum

在支持 Sealed 类之前，只能在单个类文件内通过 Enum 建模固定类型，Sealed 则更加灵活，而且能够用于模式匹配。

Enum 类可以通过 values 方法遍历子类，而 Seaed 类的子类也可以通过 getPermittedSubclasses 来遍历。

### 模式匹配：switch

之前的 switch 表达式是有限的，仅能用于判定完全相等性，且仅支持有限的类型：数字、枚举、字符串。

该预览特性支持 switch 表达式走用于任意类型，以及更加复杂的匹配模式。

原来的模式：

```java
var symbol = switch (expression) {
  case ADDITION       -> "+";
  case SUBTRACTION    -> "-";
  case MULTIPLICATION -> "*";
  case DIVISION       -> "/";
};
```

增强后支持类型模式语法：

```java
return switch (expression) {
  case Addition expr       -> "+";
  case Subtraction expr    -> "-";
  case Multiplication expr -> "*";
  case Division expr       -> "/";
};
```

比如：

```java
String formatted = switch (o) {
    case Integer i && i > 10 -> String.format("a large Integer %d", i); // 引用了 i
    case Integer i           -> String.format("a small Integer %d", i);
    default                  -> "something else";
};
```

同时支持 null 值，不再是抛出 NPE：

```java
switch (s) {
  case null  -> System.out.println("Null");
  case "Foo" -> System.out.println("Foo");
  default    -> System.out.println("Something else");
}
```

如果匹配语句没有包含所有可能的输入，编译器则会直接报错：

```java
Object o = 1234;

// OK
String formatted = switch (o) {
    case Integer i && i > 10 -> String.format("a large Integer %d", i);
    case Integer i           -> String.format("a small Integer %d", i);
    default                  -> "something else";
};

// Compile error - 'switch' expression does not cover all possible input values
String formatted = switch (o) {
    case Integer i && i > 10 -> String.format("a large Integer %d", i);
    case Integer i           -> String.format("a small Integer %d", i);
};

// Compile error - the second case is dominated by a preceding case label
String formatted = switch (o) {
    case Integer i           -> String.format("a small Integer %d", i);
    case Integer i && i > 10 -> String.format("a large Integer %d", i);
    default                  -> "something else";
};
```

