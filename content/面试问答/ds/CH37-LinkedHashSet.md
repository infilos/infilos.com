---
type: docs
title: "CH37-LinkedHashSet"
linkTitle: "CH37-LinkedHashSet"
weight: 37
---

LinkedHashSet继承了HashSet，奇怪的是HashSet暴露了一个包可见的构造器，基于LinkedHashMap构造了支持插入顺序的集合:

- LinkedHashSet不是线程安全的，它的并发修改会导致Fail-Fast.
- LinkedHashSet基于LinkedHashMap，而且由于集合元素的不可重复性，几乎可以认为它的插入操作，判断是否包含某一个元素和删除某一个元素的操作是O(1)的.

```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {

    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }

    /**
    * Constructs a new, empty linked hash set with the specified initial
    * capacity and the default load factor (0.75).
    *
    * @param  initialCapacity  the initial capacity of the LinkedHashSet
    * @throws  IllegalArgumentException if the initial capacity is less
    *              than zero
    */
    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }

    /**
    * Constructs a new, empty linked hash set with the default initial
    * capacity (16) and load factor (0.75).
    */
    public LinkedHashSet() {
        super(16, .75f, true);
    }
}
```

HashSet奇怪的构造器:

```java
 */
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

由于LinkedHashMap在HashMap之上通过维护一个支持插入顺序的链表来实现了对插入顺序的支持，而且几乎没有性能损失. LinkedHashSet在LinkedHashMap之上也提供了支持集合的，支持顺序的set。它的效率也非常高，由于集合元素的不可重复性，一般情况下几乎可以认为它的操作都是O(1).