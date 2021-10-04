---
type: docs
title: "CH07-LinkedHashSet-Map"
linkTitle: "CH07-LinkedHashSet-Map"
weight: 7
---

## 概述

- LinkedHashSet 和 LinkedHashMap 在 Java 中也是类似的实现，前者只是对后者的简单封装。
- LinkedHashMap 实现了 Map 接口，允许放入 null key 和 null value。
- 同时满足 HashMap 和 linked list 的一些特性。
- 可以将 LinkedHashMap 看做是通过 linked list 增强的 HashMap。
- LinkedHashMap 是 HashMap 的直接子类，二者唯一的区别是 LinkedHashMap 在 HashMap 的基础上，采用双向链表的形式将所有 entry 连接起来，以保证元素的迭代顺序和插入顺序相同。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210418160718.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>



如上图，相比 HashMap，在 entry 部分多了个属性用于连接所有 entry。而 header 用于指向双向链表的头部。

这种结构体还有一个好处，迭代时不需要像 HashMap 那样遍历整个 table，只需要遍历 header 指向的双向链表即可。也就是说，LinkedHashMap 的迭代时间只和 entry 的数量相关，与 table 的大小无关。

有两个参数可以影响 LinkedHashMap 的性能：初始容量(inital capacity)和负载系数(load factor)。初始容量指定了 table 的大小，负载系数用来指定自动扩容的临界值。当`entry`的数量超过`capacity*load_factor`时，容器将自动扩容并重新哈希。对于插入元素较多的场景，将初始容量设大可以减少重新哈希的次数。

著作权归https://pdai.tech所有。 链接：https://www.pdai.tech/md/java/collection/java-map-LinkedHashMap&LinkedHashSet.html

将对象放入到*LinkedHashMap*或*LinkedHashSet*中时，有两个方法需要特别关心: `hashCode()`和`equals()`。**`hashCode()`方法决定了对象会被放到哪个`bucket`里，当多个对象的哈希值冲突时，`equals()`方法决定了这些对象是否是“同一个对象”**。所以，如果要将自定义的对象放入到`LinkedHashMap`或`LinkedHashSet`中，需要重写 `hashCode()`和`equals()`方法。

## 内部实现

### get

`get(Object key)`方法根据指定的`key`值返回对应的`value`。该方法跟`HashMap.get()`方法的流程几乎完全一样。

### put

`put(K key, V value)`方法是将指定的`key, value`对添加到`map`里。该方法首先会对`map`做一次查找，看是否包含该元组，如果已经包含则直接返回，查找过程类似于`get()`方法；如果没有找到，则会通过`addEntry(int hash, K key, V value, int bucketIndex)`方法插入新的`entry`。

注意这里的插入有两重含义：

- 从 table 的角度看，新的 entry 需要插入到对应的 bucket 中，当有哈希冲突时，采用头插法将新的 entry 插入到冲突链表的头部。
- 从 header 的角度看，新的 entry 需要插入到双向链表大尾部。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210418162108.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

addEntry 的实现逻辑：

```java
// LinkedHashMap.addEntry()
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);// 自动扩容，并重新哈希
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = hash & (table.length-1);// hash%table.length
    }
    // 1.在冲突链表头部插入新的entry
    HashMap.Entry<K,V> old = table[bucketIndex];
    Entry<K,V> e = new Entry<>(hash, key, value, old);
    table[bucketIndex] = e;
    // 2.在双向链表的尾部插入新的entry
    e.addBefore(header);
    size++;
}
```

上述代码中用到了 addBefore 方法将新的 entry 插入到双向链表头引用的 header 的前面，这样 e 就称为双向链表中的最后一个元素。addBefore 的实现逻辑如下：

```java
// LinkedHashMap.Entry.addBefor()，将this插入到existingEntry的前面
private void addBefore(Entry<K,V> existingEntry) {
    after  = existingEntry;
    before = existingEntry.before;
    before.after = this;
    after.before = this;
}
```

上述到吗只是简单的修改 entry 的引用就实现了整个逻辑。

### remove

`remove(Object key)`的作用是删除`key`值对应的`entry`，该方法的具体逻辑是在`removeEntryForKey(Object key)`里实现的。`removeEntryForKey()`方法会首先找到`key`值对应的`entry`，然后删除该`entry`(修改链表的相应引用)。查找过程跟`get()`方法类似。

注意这里的删除也有两重含义：

- 从 table 的角度看，需要将 entry 从对应的 bucket 中删除，如果对应的冲突链表不为空，需要修改冲突链表的引用。
- 从 header 的角度看，需要将该 entry 从双向链表中删除，同时修改链表中前置和后置元素的引用。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210418162444.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

removeEntryForKey 的实现逻辑如下：

```java
// LinkedHashMap.removeEntryForKey()，删除key值对应的entry
final Entry<K,V> removeEntryForKey(Object key) {
	......
	int hash = (key == null) ? 0 : hash(key);
    int i = indexFor(hash, table.length);// hash&(table.length-1)
    Entry<K,V> prev = table[i];// 得到冲突链表
    Entry<K,V> e = prev;
    while (e != null) {// 遍历冲突链表
        Entry<K,V> next = e.next;
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k)))) {// 找到要删除的entry
            modCount++; size--;
            // 1. 将e从对应bucket的冲突链表中删除
            if (prev == e) table[i] = next;
            else prev.next = next;
            // 2. 将e从双向链表中删除
            e.before.after = e.after;
            e.after.before = e.before;
            return e;
        }
        prev = e; e = next;
    }
    return e;
}
```

## LinkedHashSet

*LinkedHashSet*是对*LinkedHashMap*的简单包装，对*LinkedHashSet*的函数调用都会转换成合适的*LinkedHashMap*方法。

```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {
    ......
    // LinkedHashSet里面有一个LinkedHashMap
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
	......
    public boolean add(E e) {//简单的方法转换
        return map.put(e, PRESENT)==null;
    }
    ......
}
```

## 常用场景

*LinkedHashMap*除了可以保证迭代顺序外，还有一个非常有用的用法: 可以轻松实现一个采用了FIFO替换策略的缓存。具体说来，LinkedHashMap有一个子类方法`protected boolean removeEldestEntry(Map.Entry<K,V> eldest)`，该方法的作用是告诉Map是否要删除“最老”的Entry，所谓最老就是当前Map中最早插入的Entry，如果该方法返回`true`，最老的那个元素就会被删除。在每次插入新元素的之后LinkedHashMap会自动询问removeEldestEntry()是否要删除最老的元素。这样只需要在子类中重载该方法，当元素个数超过一定数量时让removeEldestEntry()返回true，就能够实现一个固定大小的FIFO策略的缓存。示例代码如下:

```java
class FifoCache<K,V> extends LinkedHashMap<K,v> {
  private final int size;
  public FifoCache(int size){
    this.size = size;
  }
  
  @Override
  protected boolean removeEldestEntry(Map.Entry<K,V> eldest){
    return size() > size;
  }
}
```

