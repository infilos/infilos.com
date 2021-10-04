---
type: docs
title: "CH20-Deque"
linkTitle: "CH20-Deque"
weight: 20
---

Deque 全称为 double ended queue，即双向队列，它允许在两侧插入或删除元素，同时也建议我们不要向其中插入 null 值。除此之外，其余特性则和父级 Queue 类似。Deque 大多数情况下不会限制元素的数量，但这不是必须的。

Deque 中定义的方法主要分为四部分，第一部分就如 Deque 定义所言，提供两侧插入或删除的方法。第二部分是继承自 Queue 的实现。第三部分表示如果要基于此实现一个 Stack，需要实现的方法。最后一部分是继承自 Collection 的方法。

## 双端操作

这里方法和Queue定义方式一致，但却是针对两侧插入删除的。

```java
//在队首添加元素
void addFirst(E e);
//在队首添加元素
boolean offerFirst(E e);

//在队尾添加元素
void addLast(E e);
boolean offerLast(E e);

//删除队首元素
E removeFirst();
E pollFirst();

//删除队尾元素
E removeLast();
E pollLast();

//获取队首元素
E getFirst();
E peekFirst();

//获取队尾元素
E getLast();
E peekLast();

//删除第一个事件，大多数指的是删除第一个和 o equals的元素
boolean removeFirstOccurrence(Object o);
//删除最后一个事件，大多数指的是删除最后一个和 o equals的元素
boolean removeLastOccurrence(Object o);
```

## 与 Queue 对应的方法

因为Queue遵循FIFO，所以其方法在Deque中对应关系有所改变，结合Deque的定义，我们很容易就想到它们的对应关系：

```java
//与addLast(E e)等价
boolean add(E e);

//与offerLast(E e)等价
boolean offer(E e);

//与removeFirst()等价
E remove();

//与pollFirst()等价
E poll();

//与getFirst()等价
E element();

//与peekFirst()等价
E peek();
```

## 实现 Stack

Stack仅在一侧支持插入删除操作等操作，遵循LIFO原则。

```java
//与addFirst()等价
void push(E e);

//与removeFirst()等价
E pop();
```

## 继承于 Collection 的方法

```java
//顺序是从队首到队尾
Iterator<E> iterator();

//顺序是从队尾到队首
Iterator<E> descendingIterator();
```