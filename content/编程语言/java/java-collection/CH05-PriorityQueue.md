---
type: docs
title: "CH05-PriorityQueue"
linkTitle: "CH05-PriorityQueue"
weight: 5
---

## 概览

- **优先队列的作用是能保证每次取出的元素都是队列中权值最小的**。
- **元素大小的评判可以通过元素本身的自然顺序(natural ordering)，也可以通过构造时传入的比较器**。
- Java 中 *PriorityQueue* 实现了 *Queue* 接口，不允许放入`null`元素。
- 底层结构为堆，通过完全二叉树实现的小顶堆，表示可以通过数组作为实现结构。
- *PriorityQueue *的`peek()`和`element()`操作是常数时间，`add()`, `offer()`, 无参数的`remove()`以及`poll()`方法的时间复杂度都是*log(N)*。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210417000430.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

观察上图中每个元素的索引编号，会发现父节点与子节点的编号存在联系：

- leftNo = parentNo*2+1
- rightNo = parentNo*2+2
- parentNo = (nodeNo-1)/2

通过这三个公式，可以轻易计算出某个节点的父节点以及子节点的索引编号。即可以通过数组来实现存储堆。

## 方法实现

### add & offer

两者语义相同，都是相对列中添加元素，只是 Queue 接口规定二者对插入失败是的处理方式不同，前者抛出异常，后者返回 false。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210417000855.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

新加入的元素可能会破坏小顶堆的性质，因此需要进行必要的调整。

```java
//offer(E e)
public boolean offer(E e) {
    if (e == null)//不允许放入null元素
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1);//自动扩容
    size = i + 1;
    if (i == 0)//队列原来为空，这是插入的第一个元素
        queue[0] = e;
    else
        siftUp(i, e);//调整
    return true;
}
```

扩容函数 grow 类似于 ArrayList 中的 grow 函数，申请更大空间的数组并复制数据。

`siftUp(int k, E x)` 方法用于插入元素 x 同时维持堆的特性：

```java
//siftUp()
private void siftUp(int k, E x) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;//parentNo = (nodeNo-1)/2
        Object e = queue[parent];
        if (comparator.compare(x, (E) e) >= 0)//调用比较器的比较方法
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = x;
}
```

调整过程为：从 k 指定的位置开始，将 x 逐层与当前点的 parent 进行比较并交换，直到满足 `x >= queue[parent]` 为止。

### element & peek

语义完全相同，都是获取但不删除队首元素，也就是队列中权值最小的那个元素，二者唯一的区别是当方法失败时前者抛出异常，后者返回`null`。

根据小顶堆的性质，堆顶那个元素就是全局最小的那个；由于堆用数组表示，根据下标关系，`0`下标处的那个元素既是堆顶元素。所以**直接返回数组`0`下标处的那个元素即可**。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210417001629.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### remove & poll

语义也完全相同，都是获取并删除队首元素，区别是当方法失败时前者抛出异常，后者返回`null`。

由于删除操作会改变队列的结构，为维护小顶堆的性质，需要进行必要的调整。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210417001703.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

```java
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    E result = (E) queue[0];//0下标处的那个元素就是最小的那个
    E x = (E) queue[s];
    queue[s] = null;
    if (s != 0)
        siftDown(0, x);//调整
    return result;
}
```

上述代码首先记录`0`下标处的元素，并用最后一个元素替换`0`下标位置的元素，之后调用`siftDown()`方法对堆进行调整，最后返回原来`0`下标处的那个元素(也就是最小的那个元素)。

重点是`siftDown(int k, E x)`方法，该方法的作用是**从`k`指定的位置开始，将`x`逐层向下与当前点的左右孩子中较小的那个交换，直到`x`小于或等于左右孩子中的任何一个为止**。

```java
//siftDown()
private void siftDown(int k, E x) {
    int half = size >>> 1;
    while (k < half) {
    	//首先找到左右孩子中较小的那个，记录到c里，并用child记录其下标
        int child = (k << 1) + 1;//leftNo = parentNo*2+1
        Object c = queue[child];
        int right = child + 1;
        if (right < size &&
            comparator.compare((E) c, (E) queue[right]) > 0)
            c = queue[child = right];
        if (comparator.compare(x, (E) c) <= 0)
            break;
        queue[k] = c;//然后用c取代原来的值
        k = child;
    }
    queue[k] = x;
}
```

### remove

用于删除队列中跟`o`相等的某一个元素(如果有多个相等，只删除一个)，该方法不是*Queue*接口内的方法，而是*Collection*接口的方法。由于删除操作会改变队列结构，所以要进行调整；又由于删除元素的位置可能是任意的，所以调整过程比其它函数稍加繁琐。具体来说，`remove(Object o)`可以分为2种情况: 1. 删除的是最后一个元素。直接删除即可，不需要调整。2. 删除的不是最后一个元素，从删除点开始以最后一个元素为参照调用一次`siftDown()`即可。此处不再赘述。

```java
//remove(Object o)
public boolean remove(Object o) {
	//通过遍历数组的方式找到第一个满足o.equals(queue[i])元素的下标
    int i = indexOf(o);
    if (i == -1)
        return false;
    int s = --size;
    if (s == i) //情况1
        queue[i] = null;
    else {
        E moved = (E) queue[s];
        queue[s] = null;
        siftDown(i, moved);//情况2
        ......
    }
    return true;
}
```

