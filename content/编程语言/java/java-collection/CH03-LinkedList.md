---
type: docs
title: "CH03-LinkedList"
linkTitle: "CH03-LinkedList"
weight: 3
---

## 概述

- *LinkedList* 同时实现了 *List* 接口和 *Deque* 接口，也就是说它既可以看作一个顺序容器，又可以看作一个队列(*Queue*)，同时又可以看作一个栈(*Stack*)。
- 栈或队列，现在的首选是 *ArrayDeque*，它有着比 *LinkedList* (当作栈或队列使用时)有着更好的性能。
- 所有跟下标相关的操作都是线性时间。
- 在首段或者末尾删除元素只需要常数时间。
- 为追求效率 *LinkedList* 没有实现同步(synchronized)。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210416002232.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

## 内部实现

### 数据结构

- 底层**通过双向链表实现**。
- 双向链表的每个节点用内部类 *Node* 表示。
- *LinkedList* 通过`first`和`last`引用分别指向链表的第一个和最后一个元素。
- 当链表为空的时候`first`和`last`都指向`null`。

```java
transient int size = 0;

/**
 * Pointer to first node.
 * Invariant: (first == null && last == null) ||
 *            (first.prev == null && first.item != null)
 */
transient Node<E> first;

/**
 * Pointer to last node.
 * Invariant: (first == null && last == null) ||
 *            (last.next == null && last.item != null)
 */
transient Node<E> last;

private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

### 构造函数

```java
/**
 * Constructs an empty list.
 */
public LinkedList() {
}

/**
 * Constructs a list containing the elements of the specified
 * collection, in the order they are returned by the collection's
 * iterator.
 *
 * @param  c the collection whose elements are to be placed into this list
 * @throws NullPointerException if the specified collection is null
 */
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

### getFirst, getLast

```java
著作权归https://pdai.tech所有。
链接：https://www.pdai.tech/md/java/collection/java-collection-LinkedList.html

/**
 * Returns the first element in this list.
 *
 * @return the first element in this list
 * @throws NoSuchElementException if this list is empty
 */
public E getFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}

/**
 * Returns the last element in this list.
 *
 * @return the last element in this list
 * @throws NoSuchElementException if this list is empty
 */
public E getLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return l.item;
}
```

###  removeFirest(), removeLast(), remove(e), remove(index)

remove 可以删除首个 equals 指定对象的元素，或者删除指定位置的元素。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210416002720.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### add

**`add(E e)`** 将在末尾添加元素，因为 last 指向链表的末尾元素，因此操作为常数时间，仅需修改几个相关的引用即可。

`add(int index, E element)` 是在指定位置插入元素，首选需要线性查找到具体位置，然后修改相关引用，完成操作。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210416003106.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### addAll

addAll(index, c) 实现方式并不是直接调用add(index,e)来实现，主要是因为效率的问题，另一个是fail-fast中modCount只会增加1次；

```java
/**
 * Appends all of the elements in the specified collection to the end of
 * this list, in the order that they are returned by the specified
 * collection's iterator.  The behavior of this operation is undefined if
 * the specified collection is modified while the operation is in
 * progress.  (Note that this will occur if the specified collection is
 * this list, and it's nonempty.)
 *
 * @param c collection containing elements to be added to this list
 * @return {@code true} if this list changed as a result of the call
 * @throws NullPointerException if the specified collection is null
 */
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}

/**
 * Inserts all of the elements in the specified collection into this
 * list, starting at the specified position.  Shifts the element
 * currently at that position (if any) and any subsequent elements to
 * the right (increases their indices).  The new elements will appear
 * in the list in the order that they are returned by the
 * specified collection's iterator.
 *
 * @param index index at which to insert the first element
 *              from the specified collection
 * @param c collection containing elements to be added to this list
 * @return {@code true} if this list changed as a result of the call
 * @throws IndexOutOfBoundsException {@inheritDoc}
 * @throws NullPointerException if the specified collection is null
 */
public boolean addAll(int index, Collection<? extends E> c) {
    checkPositionIndex(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;

    Node<E> pred, succ;
    if (index == size) {
        succ = null;
        pred = last;
    } else {
        succ = node(index);
        pred = succ.prev;
    }

    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }

    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
}
```

### clear

为了让GC更快可以回收放置的元素，需要将node之间的引用关系赋值为 null。

```java
/**
 * Removes all of the elements from this list.
 * The list will be empty after this call returns.
 */
public void clear() {
    // Clearing all of the links between nodes is "unnecessary", but:
    // - helps a generational GC if the discarded nodes inhabit
    //   more than one generation
    // - is sure to free memory even if there is a reachable Iterator
    for (Node<E> x = first; x != null; ) {
        Node<E> next = x.next;
        x.item = null;
        x.next = null;
        x.prev = null;
        x = next;
    }
    first = last = null;
    size = 0;
    modCount++;
}
```

### Positional Access 方法

通过 index 获取元素：

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

通过 index 赋值元素：

```java
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```

通过 index 插入元素：

```java
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```

通过 index 删除元素：

```java
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}
```

### 查找

即查找元素的下标，查找第一次出现元素值相等的 index，否则返回 -1：

```java
public int indexOf(Object o) {
    int index = 0;
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}
```

查找最后一次出现的元素则类似，区别是从 last 开始向前查找。

### Queue 方法

- peek
- element
- poll
- remove
- offer

### Deque 方法

- offerFirst
- offerLast
- peekFirst
- peekLast
- pollFirst
- pollLast
- push
- pop

