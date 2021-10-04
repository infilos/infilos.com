---
type: docs
title: "CH33-HashTable"
linkTitle: "CH33-HashTable"
weight: 33
---

HashTable是类似于HashMap的容器,但是:

- HashTable是线程安全的，几乎所有的方法都是synchronized，因此，它也是阻塞的，在高并发的情况下应该使用效率更好的容器.
- HashTable与Stack，Vector一样，也支持Fail-Fast.
- HashTable底层类似于hashMap，但没有在冲突过大导致链较长的时候转化为红黑树，因此在元素较多，冲突较大时，效率上没有HashMap高.
- HashTable也不是有序的，既不保证插入的顺序也不能够使用Comparator来保证顺序.

## HashTable构造器

与HashMap类似，HashTable也支持指定初始容量，loadFactor，扩容策略基本相同.

```java
public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new Entry<?,?>[initialCapacity];
        threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    }

    /**
     * Constructs a new, empty hashtable with the specified initial capacity
     * and default load factor (0.75).
     *
     * @param     initialCapacity   the initial capacity of the hashtable.
     * @exception IllegalArgumentException if the initial capacity is less
     *              than zero.
     */
    public Hashtable(int initialCapacity) {
        this(initialCapacity, 0.75f);
    }

    /**
     * Constructs a new, empty hashtable with a default initial capacity (11)
     * and load factor (0.75).
     */
    public Hashtable() {
        this(11, 0.75f);
    }
```

HashTable每次在添加元素以后，会判断其目前已经添加的元素数量是否达到了一个阈值threshold，是的话就会扩容，然后rehash:

```java
protected void rehash() {
  int oldCapacity = table.length;
  Entry<?,?>[] oldMap = table;

  // overflow-conscious code
  int newCapacity = (oldCapacity << 1) + 1;
  if (newCapacity - MAX_ARRAY_SIZE > 0) {
      if (oldCapacity == MAX_ARRAY_SIZE)
          // Keep running with MAX_ARRAY_SIZE buckets
          return;
      newCapacity = MAX_ARRAY_SIZE;
  }
  Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

  modCount++;
  threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
  table = newMap;

  for (int i = oldCapacity ; i-- > 0 ;) {
      for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
          Entry<K,V> e = old;
          old = old.next;

          int index = (e.hash & 0x7FFFFFFF) % newCapacity;
          e.next = (Entry<K,V>)newMap[index];
          newMap[index] = e;
      }
  }
```

rehash是一个相当耗时的工作O(n)，它涉及到重新分配数组，为原数组中所有元素计算hash值.

## 插入删除，获取元素

HashTable支持线程安全的插入，删除，获取元素，由于基于hashtable，在hash比较均匀的情况下，效率很高:

```java
public synchronized V put(K key, V value) {
  // Make sure the value is not null
  if (value == null) {
      throw new NullPointerException();
  }

  // Makes sure the key is not already in the hashtable.
  Entry<?,?> tab[] = table;
  int hash = key.hashCode();
  int index = (hash & 0x7FFFFFFF) % tab.length;
  @SuppressWarnings("unchecked")
  Entry<K,V> entry = (Entry<K,V>)tab[index];
  for(; entry != null ; entry = entry.next) {
      if ((entry.hash == hash) && entry.key.equals(key)) {
          V old = entry.value;
          entry.value = value;
          return old;
      }
  }

  addEntry(hash, key, value, index);
  return null;
}
public synchronized V put(K key, V value) {
  // Make sure the value is not null
  if (value == null) {
      throw new NullPointerException();
  }

  // Makes sure the key is not already in the hashtable.
  Entry<?,?> tab[] = table;
  int hash = key.hashCode();
  int index = (hash & 0x7FFFFFFF) % tab.length;
  @SuppressWarnings("unchecked")
  Entry<K,V> entry = (Entry<K,V>)tab[index];
  for(; entry != null ; entry = entry.next) {
      if ((entry.hash == hash) && entry.key.equals(key)) {
          V old = entry.value;
          entry.value = value;
          return old;
      }
  }

  addEntry(hash, key, value, index);
  return null;
}
public synchronized V get(Object key) {
  Entry<?,?> tab[] = table;
  int hash = key.hashCode();
  int index = (hash & 0x7FFFFFFF) % tab.length;
  for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
      if ((e.hash == hash) && e.key.equals(key)) {
          return (V)e.value;
      }
  }
  return null;
}
```