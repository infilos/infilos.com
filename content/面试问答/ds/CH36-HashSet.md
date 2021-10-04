---
type: docs
title: "CH36-HashSet"
linkTitle: "CH36-HashSet"
weight: 36
---

HashSet底层依据HashMap来实现了基于Hash的集合。HashSet基本上是HashMap的一层封装.

- HashSet并不是线程安全的，HashSet支持集合的特性，每个元素会作为HashMap的键来保存，从而排斥重复。HashSet提供了集合基本的插入，删除，遍历等方法。
- HashSet也支持Fail-Fast.

HashSet由于基于HashMap，因此它的初始化和HashMap类似，支持指定初始容量和loadFactor。

## HashSet初始化

```java
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

/**
 * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
 * the specified initial capacity and default load factor (0.75).
 *
 * @param      initialCapacity   the initial capacity of the hash table
 * @throws     IllegalArgumentException if the initial capacity is less
 *             than zero
 */
public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}
```

HashSet基于HashMap，因此，可以非常方便的实现，由于集合元素的特性，它得不可重复性在大多数情况下保证了key不会冲突。因此，你可以认为它得时间复杂度是O(1)的。

## 插入删除元素

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}

public boolean contains(Object o) {
    return map.containsKey(o);
}

public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```
