---
type: docs
title: "CH35-EnumSet"
linkTitle: "CH35-EnumSet"
weight: 35
---

EnumSet用于持有Enum类型的值，由于底层使用位数组来支持，因此非常高效:

- EnumSet不是一个有序的集合，他的遍历也不是按照元素添加的循序.
- EnumSet不是线程安全的
- EnumSet也不支持Fail-Fast

EnumSet非常适用于需要持有很多枚举元素的容器，并且支持快速的添加，删除，判断操作。

EnumSet是一个抽象类，没有提供构造器，创建一个EnumSet类似于工厂方法可以这样调用:

```java
public static <E extends Enum<E>> EnumSet<E> allOf(Class<E> elementType) {
    EnumSet<E> result = noneOf(elementType);
    result.addAll();
    return result;
}
public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
    Enum<?>[] universe = getUniverse(elementType);
    if (universe == null)
        throw new ClassCastException(elementType + " not an enum");

    if (universe.length <= 64)
        return new RegularEnumSet<>(elementType, universe);
    else
        return new JumboEnumSet<>(elementType, universe);
}
private static <E extends Enum<E>> E[] getUniverse(Class<E> elementType) {
    return SharedSecrets.getJavaLangAccess()
                                    .getEnumConstantsShared(elementType);
}
```

让我们首先感觉有点惊奇的是: allOf只是指明了类的类型，然后用这个类型获取了enum类型所有的值，底层使用的是sun.misc.SharedSecrets这个类，这个类提供了一些很有趣的方法。

RegularEnumSet和JumboEnumSet就是EnumSet这个类的具体实现。EnumSet也提供了从有限的元素中初始化:

```java
public static <E extends Enum<E>> EnumSet<E> of(E first, E… rest) {
    EnumSet<E> result = noneOf(first.getDeclaringClass());
    result.add(first);
    for (E e : rest)
        result.add(e);
    return result;
}
```

EnumSet也支持add等方法，添加枚举元素到集合中.

## 实现原理

EnumSet非常高效，内部使用了bit来存储Enum元素对应的整形值. java的枚举类型的每个元素除了一个关联的名字以外，通常还有一个整形值，对应这个元素在枚举类型的所有元素组成的数组的下标，从0开始.

EnumSet内部保存的不是对应的元素，而是枚举元素对应的下标值。

EnumSet内部初始化时，具体的实现分为RegularEnumSet和JumboEnumSet, RegularEnumSet适合于元素个数小于等于64的枚举类型，其内部使用了一个long来保存所有的元素.

### RegularEnumSet

RegularEnumSet构造器常常以枚举类型和枚举类型的所有值来初始化:

```java
RegularEnumSet(Class<E>elementType, Enum<?>[] universe) {
    super(elementType, universe);
}
```

当add一个元素的时候:

```java
public boolean add(E e) {
    typeCheck(e);

    long oldElements = elements;
    elements |= (1L << ((Enum<?>)e).ordinal());
    return elements != oldElements;
}
```

这里的elements是long类型，初始值为0. 用于保存所有已添加的元素对应的下标。这里通过一个简单的位操作，将元素下标，在elements对应的位置置1:

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221141549.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

当清空所有元素的时候，直接设置elements为0即可

```java
public void clear() {
    elements = 0;
}
```

当判断是否存在某个元素的时候，可以直接判断对应的位置是否为1，即可判断是否存在:

```java
public boolean remove(Object e) {
    if (e == null)
        return false;
    Class<?> eClass = e.getClass();
    if (eClass != elementType && eClass.getSuperclass() != elementType)
        return false;

    long oldElements = elements;
    elements &= ~(1L << ((Enum<?>)e).ordinal());
    return elements != oldElements;
}
```

也是只需要简单的位操作即可。

RegularEnumSet的遍历也相当简单:

```java
public E next() {
    if (unseen == 0)
        throw new NoSuchElementException();
    lastReturned = unseen & -unseen;
    unseen -= lastReturned;
    return (E) universe[Long.numberOfTrailingZeros(lastReturned)];
}
```

RegularEnumSet对应的Iterator只需要每次获得对应的1位，从而获得对应的元素.

### JumboEnumSet

当枚举类型元素的数量大于64的时候，就会创建一个JumboEnumSet. JumboEnumSet内部使用了long的数组来保留所有的元素，（很难想象一个枚举类型元素个数超过64个的场景):

```java
JumboEnumSet(Class<E>elementType, Enum<?>[] universe) {
  super(elementType, universe);
  elements = new long[(universe.length + 63) >>> 6];
}
```

long数组可以理解为一个桶，每个桶里可以放置64个元素。由于元素个数很大，就分别放在不同的桶里。桶的数量=(元素个数 + 63) / 64，加63是为了保证有足够的桶.

每当添加一个元素的时候，首先计算桶的位置，然后使用和RegularEnumSet相同的方法来存放:

```java
public boolean add(E e) {
  typeCheck(e);

  int eOrdinal = e.ordinal();
  int eWordNum = eOrdinal >>> 6;

  long oldElements = elements[eWordNum];
  elements[eWordNum] |= (1L << eOrdinal);
  boolean result = (elements[eWordNum] != oldElements);
  if (result)
      size++;
  return result;
}
```