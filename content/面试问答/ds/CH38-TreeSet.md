---
type: docs
title: "CH38-TreeSet"
linkTitle: "CH38-TreeSet"
weight: 38
---

TreeSet是一个有序的集合，它支持特定的排序方法。TreeSet底层基于TreeMap实现，在它支持集合基本的方法外，他还保证了插入的顺序，而由于这种顺序的特性，它甚至提供了一种类似于双端队列的接口方法，支持从头部尾部取出元素，返回某个返回的子集合.

- TreeSet不是线程安全的，它的并发修改会导致Fail-Fast
- TreeSet底层基于TreeMap，这种红黑树的结构保证了插入，删除，获取某个元素的复杂度都为0(log n).

## TreeSet构造器

与TreeMap类似，TreeSet的构造器支持提供一个比较器Comparator，如果没有，相应的元素应该实现Comparator接口:

```java
public TreeSet() {
    this(new TreeMap<E,Object>());
}

/**
 * Constructs a new, empty tree set, sorted according to the specified
 * comparator.  All elements inserted into the set must be <i>mutually
 * comparable</i> by the specified comparator: {@code comparator.compare(e1,
 * e2)} must not throw a {@code ClassCastException} for any elements
 * {@code e1} and {@code e2} in the set.  If the user attempts to add
 * an element to the set that violates this constraint, the
 * {@code add} call will throw a {@code ClassCastException}.
 *
 * @param comparator the comparator that will be used to order this set.
 *        If {@code null}, the {@linkplain Comparable natural
 *        ordering} of the elements will be used.
 */
public TreeSet(Comparator<? super E> comparator) {
    this(new TreeMap<>(comparator));
}
```

TreeSet与TreeMap一样，在支持add等操作之外，还支持返回集合的某个子集，支持返回头部尾部元素，而这正是红黑树的好处:

## 查询方法

```java
public E first() {
    return m.firstKey();
}

/**
 * @throws NoSuchElementException {@inheritDoc}
 */
public E last() {
    return m.lastKey();
}

public E pollFirst() {
    Map.Entry<E,?> e = m.pollFirstEntry();
    return (e == null) ? null : e.getKey();
}

/**
 * @since 1.6
 */
public E pollLast() {
    Map.Entry<E,?> e = m.pollLastEntry();
    return (e == null) ? null : e.getKey();
}
```