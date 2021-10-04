---
type: docs
title: "CH07-Final"
linkTitle: "CH07-Final"
weight: 7
---

## 基本用法

### 修饰类

当某个类的整体定义为 final 时，就表明了你不能打算继承该类，而且也不允许别人这么做。即这个类是不能有子类的。

final 类中的所有方法都隐式为 final，因为无法覆写他们，所以在 final 类中给任何方法添加 final 关键字是没有任何意义的。

### 修饰方法

- private 方法是隐式的 final，即不能被子类重写
- final 方法是可以被重载的

#### private final

类中所有 private 方法都隐式地指定为 final 的，由于无法取用 private 方法，所以也就不能覆盖它。可以对 private 方法增添 final 关键字，但这样做并没有什么好处。

#### final 方法可以被重载

### 修饰参数

Java 允许在参数列表中以声明的方式将参数指明为 final，这意味这你无法在方法中更改参数引用所指向的对象。这个特性主要用来向匿名内部类传递数据。

### 修饰字段

#### 并非所有的 fianl 字段都是编译期常量

比如：

```java
class Example {
  Random random = new Random();
  final int value = random.nextInt();
}
```

这里的字段 value 并不能在编译期推导出实际的值，而是在运行时由 random 决定。

#### static final

static final 字段只是占用一段不能改变的存储空间，它必须在定义的时候进行赋值，否则编译期无法同步。

#### blank final

Java 允许生成空白 final，也就是说被声明为 final 但又没有给出定值的字段，但是必须在该字段被使用之前被赋值，这给予我们两种选择：

- 在定义处进行赋值(这不是空白 final)
- 在构造器中进行赋值，保证了该值在被使用之前赋值。

## 重排序规则

### final 域为基本类型

```java
public class FinalDemo {
    private int a;  										//普通域
    private final int b; 								//final域
    private static FinalDemo finalDemo; //静态域

    public FinalDemo() {
        a = 1; // 1. 写普通域
        b = 2; // 2. 写final域
    }

    public static void writer() {
        finalDemo = new FinalDemo();
    }

    public static void reader() {
        FinalDemo demo = finalDemo; // 3.读对象引用
        int a = demo.a;    //4.读普通域
        int b = demo.b;    //5.读final域
    }
}
```

假设线程 A 执行 writer 方法，线程 B 执行 reader 方法。

#### 写操作

写 final 域的重排序规则禁止对 final 域的写操作重排序到构造函数之外，该规则的实现主要包含两个方面：

- JMM 禁止编译器把 final 域的写重排序到构造函数之外。
- 编译器会在 final 域写之后，构造函数 return 之前，插入一个 storestore 屏障。
  - 该屏障可以禁止处理器将 final 域的写重排序到构造函数之外。

writer 方法分析：

- 构造了一个 FinalDemo 对象。
- 把这个对象复制给成员变量 finalDemo。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210425210946.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

由于 a，b 之间没有依赖，普通域 a 可能会被重排序到构造函数之外，线程 B 肯呢个读到普通变量 a 初始化之前的值(零值)，即引起错误。

而 final 域变量 b，根据重排序规则，会禁止 final 修饰的变量 b 被重排序到构造函数之外，因此 b 会在构造函数内完成赋值，线程 B 可以读到正确赋值后的 b 变量。

因此，写 final 域的重排序规则可以确保：在对象引用被任意线程可见之前，对象的 final 域已经被正确初始化过了，而普通域就不具有这个保障。

#### 读操作

读 final 域的重排序规则为：在一个线程中，初次读对象引用和初次读该对象包含的 final 域，JMM 会禁止这两个操作的重排序。(仅针对处理器)，处理器会在读 final 域操作之前插入一个 LoadLoad 屏障。

实际上，度对象的引用和读对象的 final 域存在间接依赖性，一般处理器不会对这两个操作执行重排序。但是不能排除有些处理器会执行重排序，因此，该规则就是针对这些处理器设定的。

reader 方法分析：

- 初次读引用变量 finalDemo；
- 初次读引用变量 finalDemo 的普通域；
- 初次读引用变量 finalDemo 的 fianl 域 b；

假设线程A写过程没有重排序，那么线程A和线程B有一种的可能执行时序为下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210425211720.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

读对象的普通域被排序到读对象引用之前，就会出现线程 B 还未多读到对象引用就在读取该对象的普票域变量，这显然是错误操作。

而 final 域的读操作就限定了在读 final 域变量前就已经读到了该对象的引用，从而避免这种错误。

读 final 域的重排序规则可以保证：在读取一个对象的 fianl 域之前，一定会先读取该 final 域所属的对象引用。

### final 域为引用类型

#### 对 final 修饰对象的成员域执行写操作

针对引用数据类型，final域写针对编译器和处理器重排序增加了这样的约束：在构造函数内对一个final修饰的对象的成员域的写入，与随后在构造函数之外把这个被构造的对象的引用赋给一个引用变量，这两个操作是不能被重排序的。注意这里的是“增加”也就说前面对final基本数据类型的重排序规则在这里还是使用。这句话是比较拗口的，下面结合实例来看。

```java
public class FinalReferenceDemo {
    final int[] arrays;
    private FinalReferenceDemo finalReferenceDemo;

    public FinalReferenceDemo() {
        arrays = new int[1];  //1
        arrays[0] = 1;        //2
    }

    public void writerOne() {
        finalReferenceDemo = new FinalReferenceDemo(); //3
    }

    public void writerTwo() {
        arrays[0] = 2;  //4
    }

    public void reader() {
        if (finalReferenceDemo != null) {  //5
            int temp = finalReferenceDemo.arrays[0];  //6
        }
    }
}
```

针对上面的实例程序，线程线程A执行wirterOne方法，执行完后线程B执行writerTwo方法，然后线程C执行reader方法。下图就以这种执行时序出现的一种情况来讨论(耐心看完才有收获)。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210425212130.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

由于对final域的写禁止重排序到构造方法外，因此1和3不能被重排序。由于一个final域的引用对象的成员域写入不能与随后将这个被构造出来的对象赋给引用变量重排序，因此2和3不能重排序。

#### 对final 修饰的对象的成员域执行读操作

JMM可以确保线程C至少能看到写线程A对final引用的对象的成员域的写入，即能看下 `arrays[0] = 1`，而写线程B对数组元素的写入可能看到可能看不到。JMM不保证线程B的写入对线程C可见，线程B和线程C之间存在数据竞争，此时的结果是不可预知的。如果可见的，可使用锁或者volatile。

### final 重排序总结

- 基本数据类型
  - 禁止 final 域写与构造函数重排序，即禁止 final 域重排序到构造方法之外，从而保证该对象对所有线程可见时，该对象的 final 域全部已经初始化过。
  - 禁止初次读取该对象的引用与读取该对象 fianl 域的重排序。
- 引用数据类型
  - 相比基本数据类型增加额外规则
  - 禁止在构造函数对一个 final 修饰的对象的成员域的写入与随后将这个被构造的对象的引用复制给引用变量重排序。
  - 即：现在构造函数中完成对 final 修饰的引用类型的字段赋值，再将该引用对象整体复制给 final 修饰的变量。

## 深入理解

### 实现原理

- 写 final 域会要求编译器在 final 域写之后，构造函数返回前插入一个 StoreStore 屏障。
- 读 final 域的重排序规则会要求编译器在读 final 域的操作前插入一个 LoadLoad 屏障。

### 为什么 final 引用不能从构造函数中逸出

上面对final域写重排序规则可以确保我们在使用一个对象引用的时候该对象的final域已经在构造函数被初始化过了。

但是这里其实是有一个前提条件的，也就是：在构造函数，不能让这个被构造的对象被其他线程可见，也就是说该对象引用不能在构造函数中“逸出”。

```java
public class FinalReferenceEscapeDemo {
    private final int a;
    private FinalReferenceEscapeDemo referenceDemo;

    public FinalReferenceEscapeDemo() {
        a = 1;  //1
        referenceDemo = this; //2
    }

    public void writer() {
        new FinalReferenceEscapeDemo();
    }

    public void reader() {
        if (referenceDemo != null) {  //3
            int temp = referenceDemo.a; //4
        }
    }
}
```

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210425212940.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

假设一个线程A执行writer方法另一个线程执行reader方法。因为构造函数中操作1和2之间没有数据依赖性，1和2可以重排序，先执行了2，这个时候引用对象referenceDemo是个没有完全初始化的对象，而当线程B去读取该对象时就会出错。尽管依然满足了final域写重排序规则：在引用对象对所有线程可见时，其final域已经完全初始化成功。但是，引用对象“this”逸出，该代码依然存在线程安全的问题。

### 使用 final 的限制条件和局限性

- 当声明一个 final 成员时，必须在构造函数退出前设置它的值。
- 或者，将指向对象的成员声明为 final 只能将该引用设为不可变的，而非所指的对象。
- 如果一个对象将会在多个线程中访问并且你并没有将其成员声明为 final，则必须提供其他方式保证线程安全。
  - 比如声明成员为 volatile，使用 synchronized 或者显式 Lock 控制所有该成员的访问。

