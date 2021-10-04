---
type: docs
title: "CH32-EnumMap"
linkTitle: "CH32-EnumMap"
weight: 32
---

EnumMap用于持有key为Enum类型的map，由于枚举类型的特殊性，EnumMap内部直接使用了简单的数组来支持，这使得EnumMap的操作相当高效.

- EnumMap不是线程安全的，并发修改需要用户自己实现线程安全.
- EnumMap没有实现Fail-Fast

## EnumMap构造器

EnumMap能够通过指定key类型来初始化一个EnumMap。内部构建了一个数组，用于持有所有的枚举元素:

```java
public EnumMap(Class<K> keyType) {
    this.keyType = keyType;
    keyUniverse = getKeyUniverse(keyType);
    vals = new Object[keyUniverse.length];
}

/**
 * Creates an enum map with the same key type as the specified enum
 * map, initially containing the same mappings (if any).
 *
 * @param m the enum map from which to initialize this enum map
 * @throws NullPointerException if <tt>m</tt> is null
 */
public EnumMap(EnumMap<K, ? extends V> m) {
    keyType = m.keyType;
    keyUniverse = m.keyUniverse;
    vals = m.vals.clone();
    size = m.size;
}
```

这里的getUniverse()方法可以返回所有的枚举类型值。并且构建数组表示最多可以容纳枚举元素个数那么多的key-value对.

## 插入，获取，删除修改元素

```java
public V put(K key, V value) {
    typeCheck(key);

    int index = key.ordinal();
    Object oldValue = vals[index];
    vals[index] = maskNull(value);
    if (oldValue == null)
        size++;
    return unmaskNull(oldValue);
}
```

EnumMap的插入，删除都可以i实现的非常高效，直接使用枚举元素的字面值，然后设置对应数组上的值即可.

```java
public V remove(Object key) {
    if (!isValidKey(key))
        return null;
    int index = ((Enum<?>)key).ordinal();
    Object oldValue = vals[index];
    vals[index] = null;
    if (oldValue != null)
        size--;
    return unmaskNull(oldValue);
}
public V get(Object key) {
    return (isValidKey(key) ?
            unmaskNull(vals[((Enum<?>)key).ordinal()]) : null);
}
```

由于底层直接使用数组，因此，插入，删除，获取的时间都是O(1)的.