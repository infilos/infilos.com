---
type: docs
title: "CH15-Iterable"
linkTitle: "CH15-Iterable"
weight: 15
---

当我们想要遍历集合时，Java 为我们提供了多种选择，通常有以下三种写法：

- for 循环
  ```java
  for (int i = 0, len = strings.size(); i < len; i++) {
    System.out.println(strings.get(i));
  }
  ```
- foreach 循环
  ```java
  for (String var : strings) {
    System.out.println(var);
  }
  ```
- while iterator
  ```java
  Iterator iterator = strings.iterator();
  while (iterator.hasNext()) {
    System.out.println(iterator.next());
  }
  ```
  
- for 循环我们很熟悉了，就是根据下标来获取元素，这个特性与数组十分吻合，不熟悉的朋友可以阅读前面讲解数组的文章。
- foreach 则主要对类似链表的结构提供遍历支持，链表没有下标，所以使用 for 循环遍历会大大降低性能。
- Iterator 就是我们今天要讲述的主角，它实际上就是 foreach。

## Iterable

Iterable 是迭代器的意思，作用是为集合类提供 for-each 循环的支持。由于使用 for 循环需要通过位置获取元素，而这种获取方式仅有数组支持，其他许多数据结构，比如链表，只能通过查询获取数据，这会大大的降低效率。Iterable 就可以让不同的集合类自己提供遍历的最佳方式。

Iterable 的文档声明仅有一句：

> Implementing this interface allows an object to be the target of the "for-each loop" statement.
> 

它的作用就是为 Java 对象提供 foreach 循环，其主要方法是返回一个 Iterator 对象：

```java
Iterator<T> iterator();
```

也就是说，如果想让一个 Java 对象支持 foreach，只要实现 Iterable 接口，然后就可以像集合那样，通过 `Iterator iterator = strings.iterator()` 方式，或者使用 foreach，进行遍历了。

## Iterator

Iterator 是 foreach 遍历的主体，它的代码实现如下：

```java
// 判断一个对象集合是否还有下一个元素
boolean hasNext();

// 获取下一个元素
E next();

// 删除最后一个元素。
// 默认是不支持的，因为在很多情况下其结果不可预测，比如数据集合在此时被修改
default void remove(){...}

// 主要将每个元素作为参数发给 action 来执行特定操作
default void forEachRemaining(Consumer<? super E> action){...}
```

Iterator 还有一个子接口，是为需要双向遍历数据时准备的，在后续分析 ArrayList 和 LinkedList 时都会看到它。它主要增加了以下几个方法：

```java
// 是否有前一个元素
boolean hasPrevious();

// 获取前一个元素
E previous();

// 获取下一个元素的位置
int nextIndex();

// 获取前一个元素的位置
int previousIndex();

// 添加一个元素
void add(E e);

// 替换当前元素值
void set(E e);
```

## 总结

在 Java 中有许多特性都是通过接口来实现的，foreach 循环也是。foreach 主要是解决 for 循环依赖下标的问题，为高效遍历更多的数据结构提供了支持。如果你清楚数组和链表的区别，应该就可以回答以下问题了：for 与 foreach 有何区别，哪个更高效？