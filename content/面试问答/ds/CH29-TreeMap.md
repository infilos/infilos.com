---
type: docs
title: "CH29-TreeMap"
linkTitle: "CH29-TreeMap"
weight: 29
---

TreeMap 是红黑树的 Java 实现，对红黑树不太了解的可以查阅前面关于红黑树的介绍。红黑树能保证增、删、查等基本操作的时间复杂度为 O(lgN)。本文将对 TreeMap 的源码进行分析。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221125028.png" style="display:block;width:70%;" alt="NAME" align=center /> </div>

## Entry 定义

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;

    Entry(K key, V value, Entry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }
    // ... 省略其他方法
}
```

## 构造函数与成员变量

### 成员变量

```java
// 比较器
private final Comparator<? super K> comparator;

// 根节点
private transient Entry<K,V> root;

// 大小
private transient int size = 0;
```

### 构造函数

```java
// 默认构造，比较器采用key的自然比较顺序
public TreeMap() {
    comparator = null;
}

// 指定比较器
public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}

// 从Map集合导入初始数据
public TreeMap(Map<? extends K, ? extends V> m) {
    comparator = null;
    putAll(m);
}

// 从SortedMap导入初始数据
public TreeMap(SortedMap<K, ? extends V> m) {
    comparator = m.comparator();
    try {
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException cannotHappen) {
    } catch (ClassNotFoundException cannotHappen) {
    }
}
```

这里用到的 putAll 和 buildFromSorted 方法，在分析完增删查等重要方法之后再进行分析。

## 重要方法

### 增加一个元素

红黑树最复杂的地方就在于增删了，我们就从增加一个元素开始分析：

```java
public V put(K key, V value) {
    // 暂存根节点
    Entry<K,V> t = root;
    
    // 根节点空，就是还没有元素
    if (t == null) {
        compare(key, key); // type (and possibly null) check
        // 新建一个元素，默认颜色黑色
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }

    // 根节点不为空，有元素时的情况
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    // 初始化时指定了comparator比较器
    if (cpr != null) {
        do {
            // 把t暂存到parent中
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                // 比较小，往左侧插入
                t = t.left;
            else if (cmp > 0)
                // 比较大，往右侧插入
                t = t.right;
            else
                // 一样大，所以就是更新当前值
                return t.setValue(value);
        } while (t != null);
    }
    else {
    // 使用key的比较器，while循环原理和上述一致
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }

    // 不断的比较，找到了没有相应儿子的节点
    //（cmp<0就是没有左儿子，cmp>0就是没有右儿子）
    Entry<K,V> e = new Entry<>(key, value, parent);
    // 把数据插入
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;

    // 新插入的元素破坏了红黑树规则，需要调整
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

fixAfterInsertion 是实现的重难点，我们先看看 Java 是如何实现的，稍后会对其中出现的几种情况做对应的图示分析。

```java
private void fixAfterInsertion(Entry<K,V> x) {
    // 先把x节点染成红色，这样可以不增加黑高，简化调整问题
    x.color = RED;
    
    // 条件是父节点是红色的，且x不是root节点，
    // 因为到root节点后就走到另外的分支了，而那个分支是正确的
    while (x != null && x != root && x.parent.color == RED) {
        //x的父节点是其祖父节点的左儿子
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            // y是x的叔叔，也就是祖父节点的右儿子
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            //叔叔是红色的
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                // 调整完毕，继续向上循环
                x = parentOf(parentOf(x));
            } else {
            // 叔叔是黑色的
                if (x == rightOf(parentOf(x))) {
                    // x是右节点，以其父节点左旋
                    x = parentOf(x);
                    rotateLeft(x);
                }
                // 右旋
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            //x的父节点是其祖父节点的右儿子
            // y是其叔叔
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                //叔叔是红色的
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                // 调整完毕，继续向上循环
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    // x是左节点，以其父节点右旋
                    x = parentOf(x);
                    rotateRight(x);
                }
                //左旋
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    
    //root节点颜色为黑色
    root.color = BLACK;
}
```

左旋和右旋代码如下：

```java
// 右旋与左旋思路一致，只分析其一
// 结果相当于把p和p的儿子调换了
private void rotateLeft(Entry<K,V> p) {
    if (p != null) {
        // 取出p的右儿子
        Entry<K,V> r = p.right;
        // 然后将p的右儿子的左儿子，也就是p的左孙子变成p的右儿子
        p.right = r.left;
        if (r.left != null)
            // p的左孙子的父亲现在是p
            r.left.parent = p;

        // 然后把p的父亲，设置为p右儿子的父亲
        r.parent = p.parent;
        // 这说明p原来是root节点
        if (p.parent == null)
            root = r;
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            p.parent.right = r;
        r.left = p;
        p.parent = r;
    }
}

//和左旋类似
private void rotateRight(Entry<K,V> p) {
    // ...
}
```

### 增加元素图示

在分析红黑树的文章中，我们已经演示过如何进行插入元素，这里结合代码再演示一次。首先再看下红黑树的定义：

- 每个节点是红色或黑色。
- 根节点是黑色。
- 每个叶子节点(NIL)是黑色。(这里叶子节点，是指为空(NIL或NULL)的叶子节点)
- 如果一个节点是红色的，则它的两个儿子都是黑色的。
- 从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。

现有一棵简单的红黑树：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221125641.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

然后我们希望把一个值为7的元素插入进去。按照put方法，先把7和根节点14比较，发现7<14，就向左遍历。到6时，发现7>6，于是再和8比较，发现8是一个叶节点，所以把7插入到8的左儿子处，如下图所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221125727.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

为了不增加黑高，这里把7设置为红色。现在，这棵树已经不再是红黑树了，因为其违反了规则如果一个节点是红色的，则它的两个儿子都是黑色的。我们按照 fixAfterInsertion 的方式对其进行调整，fixAfterInsertion 中的参数x就是这里的7。

首先，进入循环后发现7的父亲是右节点，进入 else 判断，7的叔叔4是红色的，于是把4和8染为黑色，6染为红色，把x参数指向6，并进入下一次循环。如下所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221125840.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

此时x是6，其父亲10是左儿子，其叔叔18是黑色的，此时代码就会走到这里：

```java
if (x == rightOf(parentOf(x))) {
    x = parentOf(x);
    rotateLeft(x);
}
    setColor(parentOf(x), BLACK);
    setColor(parentOf(parentOf(x)), RED);
    rotateRight(parentOf(parentOf(x)));
```

此时，就需要把10和14的颜色更换，如下图所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221125938.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

然后以14为基础右旋，涉及到的元素有10、12和14，如下所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221130018.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

具体操作为，把10的右儿子12，变为14的左儿子，然后把14变为10的右儿子，结果如下：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221130103.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

此时循环条件不再满足，也就是调整完毕，可以看到，依然是一棵正确的红黑树。

这只是需要调整的一种情况，再举一个复杂一些的例子，此时把11插入了红黑树中：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221130128.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

此时其父亲10是红色，没有叔叔，所以需要先左旋，再右旋。具体操作如下：

1. 以10为基础左旋，涉及元素为10和11。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221130213.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

情况就和之前插入7类似了，更改11和12的颜色，然后x指向12：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221130240.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

这时又和刚插入11时类似，以8为基础左旋：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221130308.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

这里是不是就很熟悉了呢？最后的结果如下所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221130316.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

代码的做法和我们之前的分析如出一辙，这里再次演示的原因是加深对理论方法与实际代码间关系的理解。

### 获取元素

TreeMap 中的元素是有序的，当使用中序遍历时就可以得到一个有序的 Set 集合，所以获取元素可以采用二分法：

```java
final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
```

除了获取某个元素外，还可以获取它的前一个元素与后一个元素：

```java
// 获取前一个元素
static <K,V> Entry<K,V> predecessor(Entry<K,V> t) {
    if (t == null)
        return null;
    else if (t.left != null) {
        // t有左孩子，所以t的前一个元素是它左孩子所在的子树的最右侧叶子结点
        Entry<K,V> p = t.left;
        while (p.right != null)
            p = p.right;
        return p;
    } else {
        // t没有左孩子，所以t的前一个元素有两种情况
        // 1. t是右孩子，那它的前一个元素就是它的父结点
        // 2. t是左孩子，它的前一个元素需要向上递归，直到递归到下一个是右孩子的节点，转为情况1
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        while (p != null && ch == p.left) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}

// 获取后一个元素
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    //...
}
```

### 删除一个元素

从红黑树中删除一个元素，和增加一个元素一样复杂。我们看看 Java 的实现：

```java
public V remove(Object key) {
    // 先用二分法获取这个元素，如果为null，不需要继续了
    Entry<K,V> p = getEntry(key);
    if (p == null)
        return null;

    V oldValue = p.value;
    deleteEntry(p);
    return oldValue;
}
```

```java
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    //如果p有两个儿子，就把p指向它的后继者，也就是它后边的元素
    if (p.left != null && p.right != null) {
        Entry<K,V> s = successor(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    // p有一个儿子，或者没有儿子，获取到之后放在replacement中
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);

    // p有儿子
    if (replacement != null) {
        // Link replacement to parent
        // 把p的子孙接在p的父级
        replacement.parent = p.parent;
        
        //p是根节点 
        if (p.parent == null)
            root = replacement;
        //p是左儿子
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        // p是右儿子
        else
            p.parent.right = replacement;

        //把p的链接都删掉
        // Null out links so they are OK to use by fixAfterDeletion.
        p.left = p.right = p.parent = null;

        // Fix replacement
        if (p.color == BLACK)
            //修正
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // return if we are the only node.
        root = null;
    } else {
        //p没有儿子
        //  No children. Use self as phantom replacement and unlink.
        if (p.color == BLACK)
            fixAfterDeletion(p);
        // 把其父节点链接到p的都去掉
        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
```

修正的方法如下所示：

```java
private void fixAfterDeletion(Entry<K,V> x) {
    while (x != root && colorOf(x) == BLACK) {
        // x是左儿子
        if (x == leftOf(parentOf(x))) {
            // sib是x的兄弟
            Entry<K,V> sib = rightOf(parentOf(x));
            
            // 兄弟是红色的
            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateLeft(parentOf(x));
                sib = rightOf(parentOf(x));
            }
            
            // 兄弟没有孩子或者孩子是黑色的
            if (colorOf(leftOf(sib))  == BLACK &&
                colorOf(rightOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                // 兄弟的右孩子是黑色的
                if (colorOf(rightOf(sib)) == BLACK) {
                    setColor(leftOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateRight(sib);
                    sib = rightOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(rightOf(sib), BLACK);
                rotateLeft(parentOf(x));
                x = root;
            }
        } else { // symmetric
            Entry<K,V> sib = leftOf(parentOf(x));

            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateRight(parentOf(x));
                sib = leftOf(parentOf(x));
            }

            if (colorOf(rightOf(sib)) == BLACK &&
                colorOf(leftOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateLeft(sib);
                    sib = leftOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(leftOf(sib), BLACK);
                rotateRight(parentOf(x));
                x = root;
            }
        }
    }

    setColor(x, BLACK);
}
```

删除元素的过程相对简单些，在分析红黑树的文章里已经做了示例，这里就不再画图展示了。

## 遗留问题

在前面分析构造函数时，有两个函数 putAll 和 buildFromSorted 当时忽略了，现在我们来看看它们的实现。

```java
public void putAll(Map<? extends K, ? extends V> map) {
	int mapSize = map.size();
	if (size==0 && mapSize!=0 && map instanceof SortedMap) {
		//...
		buildFromSorted(
			mapSize,map.entrySet().iterator(),null, null);
    	//...
		return;
     }
     super.putAll(map);
}
```

putAll 当 Map 是一个 SortedMap 实例时，依赖于 buildFromSorted，其他情况则是由 AbstractMap 实现的。所以这里重点看下 buildFromSorted 的实现。

buildFromSorted 有两个，一个是供 putAll 等调用的，另外一个则是具体的实现。

```java
// 这个方法主要是被调用，关注它只为了看下computeRedLevel这个方法
private void buildFromSorted(int size, Iterator<?> it,
                                 java.io.ObjectInputStream str,
                                 V defaultVal)
        throws java.io.IOException, ClassNotFoundException {
    this.size = size;
    root = buildFromSorted(0, 0, size - 1, computeRedLevel(size),
                it, str, defaultVal);
}
```

这里调用了一个 computeRedLevel 的方法，是这里的关键。

这个方法和染色为红色有关，其实现和二分法看似有一定联系，其文档说明它是：

> Find the level down to which to assign all nodes BLACK.  This is the last 'full' level of the complete binary tree produced by buildTree. The remaining nodes are colored RED. (This makes a `nice' set of color assignments wrt future insertions.) This level number is computed by finding the number of splits needed to reach the zeroeth node.  (The answer is ~lg(N), but in any case must be computed by same quick O(lg(N)) loop.)

大概意思是讲通过这种方式可以构建一个优秀的红黑树，能够为以后插入更多数据提供便利。

最后我们看下 buildFromSorted 的实现：

```java
private final Entry<K,V> buildFromSorted(int level, int lo, int hi,
                                         int redLevel,
                                         Iterator<?> it,
                                         java.io.ObjectInputStream str,
                                         V defaultVal)
    throws  java.io.IOException, ClassNotFoundException {
 
    if (hi < lo) return null;
    
    // 获取中间位置
    int mid = (lo + hi) >>> 1;

    Entry<K,V> left  = null;
    if (lo < mid)
        // 递归左子树，和压栈类似，直到lo>=mid才能返回结果
        left = buildFromSorted(level+1, lo, mid - 1, redLevel,
                                it, str, defaultVal);

    // extract key and/or value from iterator or stream
    K key;
    V value;
    if (it != null) {
        // 给key和value赋值
        if (defaultVal==null) {
            Map.Entry<?,?> entry = (Map.Entry<?,?>)it.next();
            key = (K)entry.getKey();
            value = (V)entry.getValue();
        } else {
            key = (K)it.next();
            value = defaultVal;
        }
    } else { // use stream
        // 从序列化中恢复
        key = (K) str.readObject();
        value = (defaultVal != null ? defaultVal : (V) str.readObject());
    }

    Entry<K,V> middle =  new Entry<>(key, value, null);

    // color nodes in non-full bottommost level red
    // 
    if (level == redLevel)
        middle.color = RED;

    if (left != null) {
        middle.left = left;
        left.parent = middle;
    }

    if (mid < hi) {
        Entry<K,V> right = buildFromSorted(level+1, mid+1, hi, redLevel,
                                            it, str, defaultVal);
        middle.right = right;
        right.parent = middle;
    }

    return middle;
}
```

根据以上方式，我们测试向其中插入10条数据，其结果类似下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190221131003.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

可见，redLevel 控制的是红色节点出现的层级，使插入的数据更整齐，方便后续操作。