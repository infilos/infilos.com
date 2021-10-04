---
type: docs
title: "CH22-PriorityQueue"
linkTitle: "CH22-PriorityQueue"
weight: 22
---

PriorityQueue（优先级队列)是一种在队列的基础上支持优先级的，PriorityQueue的优先级表现为一个PriorityQueue会关联一个Comparator，Comparator的结果体现了优先级的大小。PriorityQueue内部使用了数组来保存元素，支持动态扩容，整个数组会当做一个堆，每次加入删除元素的时候会调整堆。

## PriorityQueue构造器

```java
public PriorityQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}
public PriorityQueue(int initialCapacity) {
    this(initialCapacity, null);
}
public PriorityQueue(int initialCapacity) {
    this(initialCapacity, null);
}
public PriorityQueue(int initialCapacity,
                     Comparator<? super E> comparator) {
    // Note: This restriction of at least one is not actually needed,
    // but continues for 1.5 compatibility
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}
```

默认的初始容量为11，如果没有提供自己的Comparator，那么会默认认为PriorityQueue的泛型类实现了Comparable接口. 初始容量为11？ 也不知道怎么想的。

## PriorityQueue的增加，删除，获取元素

```java
public boolean add(E e) {
    return offer(e);
}

public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1);
    size = i + 1;
    if (i == 0)
        queue[0] = e;
    else
        siftUp(i, e);
    return true;
}
```

PriorityQueue的动态增长策略是:

```java
private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    // Double size if small; else grow by 50%
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                     (oldCapacity + 2) :
                                     (oldCapacity >> 1));
    // overflow-conscious code
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    queue = Arrays.copyOf(queue, newCapacity);
}
```

原长度小于64，就每次只加2，大于64之后，就在原来的基础上增加1.5倍。what the hell… 
PriorityQueue底层的数组，每次在插入，移除某个元素时都需要重新调整堆:

```java
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    E result = (E) queue[0];
    E x = (E) queue[s];
    queue[s] = null;
    if (s != 0)
        siftDown(0, x);
    return result;
}

private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}

@SuppressWarnings("unchecked")
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>)x;
    int half = size >>> 1;        // loop while a non-leaf
    while (k < half) {
        int child = (k << 1) + 1; // assume left child is least
        Object c = queue[child];
        int right = child + 1;
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        if (key.compareTo((E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = key;
}
```

这种堆在优先级上来说，是最大堆，数组的0位置元素是最优先的，这个最优先的元素在Comparator比较小，具有最小的值，这种越小的值越优先的策略可以理解为对弱者的一种重视.

## Others

PriorityQueue这种基于堆的结构，在插入和删除是都非常高效，O(logn)。PriorityQueue也支持序列化。提供了readObject和writeObject方法，这两个方法在ObjectInputStream和ObjectOutputStream时都会调用.