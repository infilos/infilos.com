---
type: docs
title: "CH26-Map"
linkTitle: "CH26-Map"
weight: 26
---

数组与链表在处理数据时各有优缺点，数组查询速度很快而插入很慢，链表在插入时表现优秀但查询无力。哈希表则整合了数组与链表的优点，能在插入和查找等方面都有不错的速度。我们之后要分析的 HashMap 就是基于哈希表实现的，不过在 JDK1.8 中还引入了红黑树，其性能进一步提升了。本文主要分析 JDK 中关于 Map 的定义。

## Map 接口

> An object that maps keys to values. A map cannot contain duplicate keys; each key can map to at most one value.
> 

也就是基于 key-value 的数据格式，并且 key 值不可以重复，每个 key 对应的 value 唯一。Map 的 key 也可以为 null，也不可重。

在分析其定义的方法前，我们要先了解一下 Map.Entry 这个接口。

## Map.Entry 接口

存储在 Map 中的数据需要实现此接口，主要提供对 key 和 value 的操作，也是我们使用最多的操作。我们先分析它：

```java
// 获取对应的key
K getKey();

// 获取对应的value
V getValue();

// 替换原有的value
V setValue(V value);

// 希望我们实现equals和hashCode
boolean equals(Object o);
int hashCode();

// 从1.8起，还提供了比较的方法，类似的方法共四个
public static <K extends Comparable<? super K>, V> Comparator<Map.Entry<K,V>> comparingByKey() {
        return (Comparator<Map.Entry<K, V>> & Serializable)
            (c1, c2) -> c1.getKey().compareTo(c2.getKey());
}
```

## 重要方法

```java
// 返回当前数据个数
int size();

// 是否为空
boolean isEmpty();

// 判断是否包含key，这里用到了key的equals方法，所以key必须实现它
boolean containsKey(Object key);

// 判断是否有key保存的值是value，这也基于equals方法
boolean containsValue(Object value);

// 通过key获取对应的value值
V get(Object key);

// 存入key-value
V put(K key, V value);

// 移除一个key-value对
V remove(Object key);

// 从其他Map添加
void putAll(Map<? extends K, ? extends V> m);

// 清空
void clear();

// 返回所有的key至Set集合中，因为key是不可重的，Set也是不可重的
Set<K> keySet();

// 返回所有的values
Collection<V> values();

// 返回key-value对到Set中
Set<Map.Entry<K, V>> entrySet();

// 希望我们实现equals和hashCode
boolean equals(Object o);
int hashCode();
```

此外，还有一些 Java8 相关的 default 方法，就不一一展示了。

```java
default V getOrDefault(Object key, V defaultValue) {
    V v;
    return (((v = get(key)) != null) || containsKey(key))
        ? v
        : defaultValue;
}
```

## 超级实现类——AbstractMap

对应于 AbstractCollection，AbstractMap 的作用也是类似的，主要是提供一些方法的实现，可以方便继承。下面我们看看它都实现了哪些方法：

```java
// 返回大小，这里大小基于entrySet的大小
public int size() {
    return entrySet().size();
}

public boolean isEmpty() {
    return size() == 0;
}

//基于entrySet操作
public boolean containsKey(Object key) {
        Iterator<Map.Entry<K,V>> i = entrySet().iterator();
        if (key==null) {
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                if (e.getKey()==null)
                    return true;
            }
        } else {
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                if (key.equals(e.getKey()))
                    return true;
            }
        }
        return false;
    }

public boolean containsValue(Object value) {
    //...
}

public V get(Object key) {
    //...
}

public V remove(Object key) {
    //...
}

public void clear() {
    entrySet().clear();
}
```

除此以外，还定义了两个变量：

```java
transient Set<K>        keySet;
transient Collection<V> values;
```

还提供了默认的实现方法，我们只看其中一个吧：

```java
public Set<K> keySet() {
    Set<K> ks = keySet;
    if (ks == null) {
        ks = new AbstractSet<K>() {
            public Iterator<K> iterator() {
                return new Iterator<K>() {
                    private Iterator<Entry<K,V>> i = entrySet().iterator();

                    public boolean hasNext() {
                        return i.hasNext();
                    }

                    public K next() {
                        return i.next().getKey();
                    }

                    public void remove() {
                        i.remove();
                    }
                };
            }

            public int size() {
                return AbstractMap.this.size();
           }

            public boolean isEmpty() {
               return AbstractMap.this.isEmpty();
            }

            public void clear() {
                AbstractMap.this.clear();
            }

            public boolean contains(Object k) {
                return AbstractMap.this.containsKey(k);
            }
        };
        keySet = ks;
    }
    return ks;
}
```

除了以上相关方法以外，AbstractMap 还实现了 equals、hashCode、toString、clone 等方法，这样在具体实现时可以省去很多工作。