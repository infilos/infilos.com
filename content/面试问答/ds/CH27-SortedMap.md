---
type: docs
title: "CH27-SortedMap"
linkTitle: "CH27-SortedMap"
weight: 27
---

由于乱序的数据对查找不利，例如无法使用二分法等降低算法的时间复杂度，如果数据在插入时就排好顺序，查找的性能就会提升很多。SortedMap 接口就是为这种有序数据服务的。

SortedMap 接口需要数据的 key 支持 Comparable，或者可以被指定的 Comparator 接受。SortedMap 主要提供了以下方法：

```java
// 返回排序数据所用的Comparator
Comparator<? super K> comparator();

// 返回在[fromKey, toKey)之间的数据
SortedMap<K,V> subMap(K fromKey, K toKey);

// 返回从第一个元素到toKey之间的数据
SortedMap<K,V> headMap(K toKey);

// 返回从fromKey到末尾之间的数据
SortedMap<K,V> tailMap(K fromKey);

//返回第一个数据的key
K firstKey();

//返回最后一个数据的key
K lastKey();
```

SortedMap 主要提供了获取子集，以及获取最大值（最后一个值）和最小值（第一个值）的方法。但这仅仅是排序数据能提供的便利的一小部分，在之后分析的 NavigableMap 中，我们还会看到更多的功能。