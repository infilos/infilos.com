---
type: docs
title: "CH09-WeakHashMap"
linkTitle: "CH09-WeakHashMap"
weight: 9
---

## 概述

它的特殊之处在于 *WeakHashMap* 里的`entry`可能会被GC自动删除，即使程序员没有调用`remove()`或者`clear()`方法。

当使用 *WeakHashMap* 时，即使没有显式的添加或删除任何元素，也可能发生如下情况:

- 调用两次 size 方法所返回的结果不同。
- 调用两次 isEmpty 方法，第一次返回 false，第二次返回 true。
- 调用两次 containskey 方法，首次返回 true，第二次返回 false，尽管两次使用相同的 key。
- 调用两次 get 方法，首次返回 value，第二次返回 null，尽管两次使用相同的对象。

这些特性尤其适用于需要缓存的场景。在缓存场景中，由于内存的局限，不能缓存所有对象，对象缓存命中可以提供系统效率，但缓存 MISS 也不会引起错误，因为可以通过计算重新得到。

Java 内存是通过 GC 自动管理的，GC 会在程序运行过程中自动判断哪些对象是可以被回收的，并在合适的时机执行内存释放。GC 判断某个对象释放可以被回收的依据是，释放有有效的引用指向该对象。如果没有有效引用指向该对象(即基本意味着不存在访问该对象的方式)，那么该对象就是可以被回收的。这里的有效应用并不包括弱引用。也就是说，虽然弱引用可以用来访问对象，但进行垃圾回收时弱引用并不会被考虑在内，仅有弱引用指向的对象仍然会被 GC 回收。

WeakHashMap 内部是通过弱引用来管理 entry 的，弱引用的特性应用到 WeakHashMap 上意味着什么呢？将一对 key value 放入到 WeakHashMap 中并不能避免该 key 被 GC 回收，除非在 WeakHashMap 在外还有对该 key 的强引用。

### 具体实现

类似于 HashMap 和 HashSet。

### WeakHashSet

```java
Set<Object> weakHashSet = Collections
  .newSetFromMap(new WeakHashMap<Object, Boolean>());
```

该工具方法可以直接将 Map 包装为 Set，只是对 Map 的简单封装。

```java
// Collections.newSetFromMap()用于将任何Map包装成一个Set
public static <E> Set<E> newSetFromMap(Map<E, Boolean> map) {
    return new SetFromMap<>(map);
}

private static class SetFromMap<E> extends AbstractSet<E>
    implements Set<E>, Serializable
{
    private final Map<E, Boolean> m;  // The backing map
    private transient Set<E> s;       // Its keySet
    SetFromMap(Map<E, Boolean> map) {
        if (!map.isEmpty())
            throw new IllegalArgumentException("Map is non-empty");
        m = map;
        s = map.keySet();
    }
    public void clear()               {        m.clear(); }
    public int size()                 { return m.size(); }
    public boolean isEmpty()          { return m.isEmpty(); }
    public boolean contains(Object o) { return m.containsKey(o); }
    public boolean remove(Object o)   { return m.remove(o) != null; }
    public boolean add(E e) { return m.put(e, Boolean.TRUE) == null; }
    public Iterator<E> iterator()     { return s.iterator(); }
    public Object[] toArray()         { return s.toArray(); }
    public <T> T[] toArray(T[] a)     { return s.toArray(a); }
    public String toString()          { return s.toString(); }
    public int hashCode()             { return s.hashCode(); }
    public boolean equals(Object o)   { return o == this || s.equals(o); }
    public boolean containsAll(Collection<?> c) {return s.containsAll(c);}
    public boolean removeAll(Collection<?> c)   {return s.removeAll(c);}
    public boolean retainAll(Collection<?> c)   {return s.retainAll(c);}
    // addAll is the only inherited implementation
    ......
}
```

