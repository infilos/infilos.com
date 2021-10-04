---
type: docs
title: "CH24-Vector"
linkTitle: "CH24-Vector"
weight: 24
---

Vector 和 ArrayList 非常相似，底层都是基于数组的实现，支持动态扩容。支持通过索引访问，删除，添加元素。Vector 也继承自 AbstractList，也实现了 Collections 接口。与 ArrayList 不同的是，Vector 是线程安全的容器。他的添加删除等方法都是 synchronized。但是 Vector 也支持 fail-fast。

## 构造器

```java
public Vector(int initialCapacity, int capacityIncrement) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
        this.capacityIncrement = capacityIncrement;
    }

    /**
     * Constructs an empty vector with the specified initial capacity and
     * with its capacity increment equal to zero.
     *
     * @param   initialCapacity   the initial capacity of the vector
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public Vector(int initialCapacity) {
        this(initialCapacity, 0);
    }

    /**
     * Constructs an empty vector so that its internal data array
     * has size {@code 10} and its standard capacity increment is
     * zero.
     */
    public Vector() {
        this(10);
    }

    /**
     * Constructs a vector containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection whose elements are to be placed into this
     *       vector
     * @throws NullPointerException if the specified collection is null
     * @since   1.2
     */
    public Vector(Collection<? extends E> c) {
        elementData = c.toArray();
        elementCount = elementData.length;
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, elementCount, Object[].class);
    }
```

与ArrayList类似，Vector支持初始化时指定大小，默认初始化的情况下，是一个可以容纳10个元素的数组。（这一点与ArrayList不同，ArrayList默认初始化时是空的数组).

## Vector添加，删除，访问元素

Vector支持向尾部添加元素，向指定位置添加元素，在向指定位置添加元素的时候需要移动。支持访问某个位置的元素，访问头部，尾部元素，支持删除某个位置的元素，删除指定元素(通过equals比较).

```java
public synchronized E firstElement() {
    if (elementCount == 0) {
        throw new NoSuchElementException();
    }
    return elementData(0);
}

/**
 * Returns the last component of the vector.
 *
 * @return  the last component of the vector, i.e., the component at index
 *          <code>size()&nbsp;-&nbsp;1</code>.
 * @throws NoSuchElementException if this vector is empty
 */
public synchronized E lastElement() {
    if (elementCount == 0) {
        throw new NoSuchElementException();
    }
    return elementData(elementCount - 1);
}
public synchronized E elementAt(int index) {
    if (index >= elementCount) {
        throw new ArrayIndexOutOfBoundsException(index + " >= " + elementCount);
    }

    return elementData(index);
}
public synchronized E elementAt(int index) {
    if (index >= elementCount) {
        throw new ArrayIndexOutOfBoundsException(index + " >= " + elementCount);
    }

    return elementData(index);
}
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}
public synchronized void insertElementAt(E obj, int index) {
    modCount++;
    if (index > elementCount) {
        throw new ArrayIndexOutOfBoundsException(index
                                                 + " > " + elementCount);
    }
    ensureCapacityHelper(elementCount + 1);
    System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
    elementData[index] = obj;
    elementCount++;
}
public synchronized void removeElementAt(int index) {
    modCount++;
    if (index >= elementCount) {
        throw new ArrayIndexOutOfBoundsException(index + " >= " +
                                                 elementCount);
    }
    else if (index < 0) {
        throw new ArrayIndexOutOfBoundsException(index);
    }
    int j = elementCount - index - 1;
    if (j > 0) {
        System.arraycopy(elementData, index + 1, elementData, index, j);
    }
    elementCount--;
    elementData[elementCount] = null; /* to let gc do its work */
}
```

Vector的扩容策略与ArrayList不同，ArrayList总是以1.5倍的大小来扩容，而Vector会支持用户在构造Vector时指定一个整形的incrementCapacity变量，每次在需要增加的时候，都会尽量增加指定的incrementCapacity的大小。

```java
 private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

如果增加incrementCapacity大小还不能保证放得下所有的元素，那么就增加可以放得下所有元素的大小。如果在创建Vector的时候，没有指定initialCapacity大小，initialCapacity是0，这就会导致每次增加元素都会扩容。这是相当低效的。jdk文档里已不推荐使用Vector。大概这是原因之一。

## Vector的序列化

```java
private void writeObject(java.io.ObjectOutputStream s)
      throws java.io.IOException {
  final java.io.ObjectOutputStream.PutField fields = s.putFields();
  final Object[] data;
  synchronized (this) {
      fields.put("capacityIncrement", capacityIncrement);
      fields.put("elementCount", elementCount);
      data = elementData.clone();
  }
  fields.put("elementData", data);
  s.writeFields();
}
```

Vector的序列化似乎只提供了writeObject方法，奇怪的是，这里方法在序列化的时候首先clone了所有的元素，尽管这样做可以保证同步，但是实在不确定这种方法的效率.

## Vector优化

尽管Vector的设计有一点缺点，但是Vector也尽量保证了删除元素的时候设置应用为null，来尽量保证GC。

```java
public synchronized void removeAllElements() {
  modCount++;
  // Let gc do its work
  for (int i = 0; i < elementCount; i++)
      elementData[i] = null;

  elementCount = 0;
}
```

在clean元素的时候，设置数组的所有元素为null.

## Vector 的线程安全性

如我们之前贴的方法所示，add , remove , get等都有synchronized关键字来保证线程安全。而且，Vector的遍历也通过modCount保证了fail-fast. 这种简单加锁的机制，在高并发的情况下，肯定会带来效率的影响。因此，在高并发时，还是需要使用效率更好的容器。