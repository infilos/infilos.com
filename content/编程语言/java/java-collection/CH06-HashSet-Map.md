---
type: docs
title: "CH06-HashSet-Map"
linkTitle: "CH06-HashSet-Map"
weight: 6
---

## 概述

- HashSet 与 HashMap 在 Java 内部的实现类似，前者仅仅是对后者进行了封装。
- HashMap 实现了 Map 接口，允许放入 null key 和 null value。
- 与 HashTable 的区别在于没有实现同步。
- 与 TreeMap 的区别在于不保证元素顺序。
- 采用冲突链表(Sepratate chaining with linked lists)解决哈希冲突。
  - 另一种实现是开放地址方式(Open Addressing)。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210418152835.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

如果选择合适的哈希函数，put 与 get 方法可以在常数内完成。但是在对 HashMap 执行迭代时，需要遍历整个 table 以及后边跟的冲突链表。因此对于迭代频繁的场景，不宜将 HashMap 的初始大小设置的过大。

有两个参数可以影响 HashMap 的性能：初始容量(inital capacity)和负载系数(load factor)。

初始容量指定了初始 table 的大小，负载系数用来指定自动扩容的临界值。当 entry 的数量超过 `capacity * load-factor` 时，容器将自动扩容并重新哈希。对于插入元素较多的场景，将初始容量设置较大可以减少重新哈希的次数。

将对象放入到 HashSet 和 HashMap 时，有两个方法要格外留意：hashCode 和 equals。

hashCode 方法决定了对象会被放到哪个 bucket 中，当多个对象的哈希值冲突，equals 方法决定了这些对象是否是同一个对象。因此，如果要将自定义的对象放入到 HashMap 或 HashSet，需要重写 hashCode 和 equals 方法。

## HashMap

### get

`get(Object key)`方法根据指定的`key`值返回对应的`value`，该方法调用了`getEntry(Object key)`得到相应的`entry`，然后返回`entry.getValue()`。因此`getEntry()`是算法的核心。 算法思想是首先通过`hash()`函数得到对应`bucket`的下标，然后依次遍历冲突链表，通过`key.equals(k)`方法来判断是否是要找的那个`entry`。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210418153601.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

上图中 `hash(k) & (table.length-1)` 等价于 `hash(k) % table.length`，原因是 HashMap 要求 `table.length` 均为 2 的指数，因此 `table.length -1` 就是二进制低位全是 1，跟 `hash(k)` 相与会将哈希值的高位全部抹掉，剩下的就是余数了。

```java
//getEntry()方法
final Entry<K,V> getEntry(Object key) {
	......
	int hash = (key == null) ? 0 : hash(key);
    for (Entry<K,V> e = table[hash&(table.length-1)];//得到冲突链表
         e != null; e = e.next) {//依次遍历冲突链表中的每个entry
        Object k;
        //依据equals()方法判断是否相等
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

### put

`put(K key, V value)`方法是将指定的`key, value`对添加到`map`里。该方法首先会对`map`做一次查找，看是否包含该元组，如果已经包含则直接返回，查找过程类似于`getEntry()`方法；如果没有找到，则会通过`addEntry(int hash, K key, V value, int bucketIndex)`方法插入新的`entry`，插入方式为**头插法**。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210418153949.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

```java
//addEntry()
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);//自动扩容，并重新哈希
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = hash & (table.length-1);//hash%table.length
    }
    //在冲突链表头部插入新的entry
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

### remove

`remove(Object key)`的作用是删除`key`值对应的`entry`，该方法的具体逻辑是在`removeEntryForKey(Object key)`里实现的。`removeEntryForKey()`方法会首先找到`key`值对应的`entry`，然后删除该`entry`(修改链表的相应引用)。查找过程跟`getEntry()`过程类似。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210418154043.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

```java
//removeEntryForKey()
final Entry<K,V> removeEntryForKey(Object key) {
	......
	int hash = (key == null) ? 0 : hash(key);
    int i = indexFor(hash, table.length);//hash&(table.length-1)
    Entry<K,V> prev = table[i];//得到冲突链表
    Entry<K,V> e = prev;
    while (e != null) {//遍历冲突链表
        Entry<K,V> next = e.next;
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k)))) {//找到要删除的entry
            modCount++; size--;
            if (prev == e) table[i] = next;//删除的是冲突链表的第一个entry
            else prev.next = next;
            return e;
        }
        prev = e; e = next;
    }
    return e;
}
```

## HashSet

*HashSet*是对*HashMap*的简单包装，对*HashSet*的函数调用都会转换成合适的*HashMap*方法。

```java
//HashSet是对HashMap的简单包装
public class HashSet<E>
{
	......
  //HashSet里面有一个HashMap
	private transient HashMap<E,Object> map;
    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
    public HashSet() {
        map = new HashMap<>();
    }
    ......
    public boolean add(E e) {//简单的方法转换
        return map.put(e, PRESENT)==null;
    }
    ......
}
```

