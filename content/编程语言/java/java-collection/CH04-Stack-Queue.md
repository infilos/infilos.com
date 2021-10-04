---
type: docs
title: "CH04-Stack-Queue"
linkTitle: "CH04-Stack-Queue"
weight: 4
---

## 概述

- Java 中存在 Stack 实现类，但没有提供 Queue 实现类，仅有一个 Queue 接口。
- 但是在需要使用栈时，Java 推荐的结果是更加高效的 ArrayQueue。
- 在需要使用队列时，首选是 ArrayQueue，其次是 LinkedList。

## Queue

Queue 接口继承自 Collection 接口，除了最基本的 Collection 方法之外，还额外支持 insertion、extraction、inspection 操作。这里有两组格式共 6 个方法，一组是抛出异常的实现，一组是返回值的实现(或 null)。

|         | Throws Exception | Returns special Value |
| ------- | ---------------- | --------------------- |
| Insert  | add(e)           | offer(e)              |
| Remove  | remove()         | poll()                |
| Examine | element()        | peek()                |

## Deque

Deque 是 "double ended queue"，表示双向队列，英文读作 `deck`。Deque 继承自 Queue 接口，除了支持 Queue 的方法外，还支持 insert、remove、examine 操作。

由于 Deque 是双向的，所以可以支持队列的头尾操作，同时支持两种格式共 12 个方法：

|         | First Element-Head |               | Last Element-Tail |                |
| ------- | ------------------ | ------------- | ----------------- | -------------- |
|         | Throws Exception   | Special Value | Throws Exception  | Speicial Value |
| Insert  | addFirst(e)        | offerFirst(e) | addLast(e)        | offerLast(e)   |
| Remove  | removeFirst()      | pollFirst()   | removeLast()      | pollLast()     |
| Examine | getFirst()         | peekFirst()   | getLast()         | peekLast()     |

当把 Deque 当做 FIFO 来使用时，元素是从 deque 的尾部添加，从头部进行删除。所谓 Deque 的部分方法和 Queue 是等同的。

| Queue Method | Equivalent Deque Method | 说明                                  |
| ------------ | ----------------------- | ------------------------------------- |
| add(e)       | addLast(e)              | 向队尾添加元素，失败时抛异常          |
| offer(e)     | offerLast(e)            | 向队尾添加元素，失败时返回 false      |
| remove()     | removeFirst()           | 获取并删除队首元素，失败时抛异常      |
| poll()       | pollFirst()             | 获取并删除队首元素，失败时返回 null   |
| element()    | getFirst()              | 获取但不删除队首元素，失败时抛异常    |
| peek()       | peekFirst()             | 获取但不删除队首元素，失败时返回 null |

Deque 与 Stack 的对应方法：

| Stack Method | Equivalent Deque Method | 说明                                   |
| ------------ | ----------------------- | -------------------------------------- |
| push(e)      | addFirst(e)             | 向栈顶插入元素，失败则抛出异常         |
| 无           | offerFirst(e)           | 向栈顶插入元素，失败则返回`false`      |
| pop()        | removeFirst()           | 获取并删除栈顶元素，失败则抛出异常     |
| 无           | pollFirst()             | 获取并删除栈顶元素，失败则返回`null`   |
| peek()       | peekFirst()             | 获取但不删除栈顶元素，失败则抛出异常   |
| 无           | peekFirst()             | 获取但不删除栈顶元素，失败则返回`null` |

以上操作中，除非对容量有限制，否则添加操作是不会失败的。

### ArrayDeque

ArrayDeque 底层为数组结构，为了满足可以同时在数组两端添加或删除元素，该数组必须是循环数组，即数组的任何一点都可能被看做起点或终点。

ArrayDeque 非线程安全，不能添加 null 元素。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210416234951.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

head 指向首端第一个有效元素，tail 指向尾端第一个可以插入元素的空位。

因为是循环数组，所谓 head 的位置不一定是 0，tail 的位置也不一定总是比 head 的位置大。

#### addFirst

在 Deque 首端添加元素，也就是在 head 前面添加元素，在空间足够且下标没有越界的情况下，只需要将 `elements[--head]=e` 即可。即将 head 的索引递减 1 的位置赋值为新加的元素。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210416235158.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

```java
//addFirst(E e)
public void addFirst(E e) {
    if (e == null)//不允许放入null
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;//2.下标是否越界
    if (head == tail)//1.空间是否够用
        doubleCapacity();//扩容
}
```

上述代码中可以发现，空间问题是在插入之后开始解决的，因为 tail 总是指向下一个可插入的空位，也就意味着 elements 数组至少会存在一个空位，所以插入元素时不用先考虑空间问题。

下标越界的解决方法很简单，`head = (head -1) & (elements.length -1)`即可，这段代码相当于取余，同时解决了 head 值为负的情况。因为 `elements.length` 必须是 2 的指数倍，`elements -1` 就是二进制低位全为 1，跟 `head-1` 相与之后就起到了取模的作用，如果 `head-1`为负(-1)，则相当于对其取 `elements.length` 的补码。

对于扩容函数 `doubleCapacity`，其逻辑就是申请一个更大的数组(原数据的两倍空间)，然后复制原来的元素。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210416235747.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

复制分为两次，第一次复制 head 右边的元素，第二次复制 head 左边的数据。

#### addLast

作用是在 *Deque* 的尾端插入元素，也就是在`tail`的位置插入元素，由于`tail`总是指向下一个可以插入的空位，因此只需要`elements[tail] = e;`即可。插入完成后再检查空间，如果空间已经用光，则调用`doubleCapacity()`进行扩容。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210416235929.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

#### poolFirst

作用是删除并返回 *Deque* 首端元素，也即是`head`位置处的元素。如果容器不空，只需要直接返回`elements[head]`即可，当然还需要处理下标的问题。由于`ArrayDeque`中不允许放入`null`，当`elements[head] == null`时，意味着容器为空。

#### pollLast

作用是删除并返回*Deque*尾端元素，也即是`tail`位置前面的那个元素。

#### peekFirst

作用是返回但不删除*Deque*首端元素，也即是`head`位置处的元素，直接返回`elements[head]`即可。

#### peekLast

作用是返回但不删除*Deque*尾端元素，也即是`tail`位置前面的那个元素。

