---
type: docs
title: "CH31-AllQueues"
linkTitle: "CH31-AllQueues"
weight: 31
---

| 名称                  | 类型   | 有界 | 线程安全 | 说明                         |
| --------------------- | ------ | ---- | -------- | ---------------------------- |
| Queue<E>              | 接口   | —    | —        | 顶层队列接口                 |
| BlockingQueue<E>      | 接口   | —    | —        | 阻塞队列接口                 |
| BlockingDeuque<E>     | 接口   | —    | —        | 双向阻塞队列接口             |
| Dequeu<E>             | 接口   | —    | —        | 双向队列接口                 |
| TransferQueue<E>      | 接口   | —    | —        | 传输队列接口                 |
| AbstractQueue         | 抽象类 | —    | —        | 队列抽象类                   |
| PriorityQueue         | 实现类 | N    | N        | 优先级队列                   |
| ArrayDeque            | 实现类 | N    | N        | 数组双向队列                 |
| LinkedList            | 实现类 | N    | N        | 链表对象类                   |
| ConcurrentLinkedQueue | 实现类 | N    | Y        | 链表结构并发队列             |
| ConcurrentLinkedDeque | 实现类 | N    | Y        | 链表结构双向并发队列         |
| ArrayBlockingQueue    | 实现类 | Y    | Y        | 数组结构有界阻塞队列         |
| LinkedBlockingQueue   | 实现类 | Y    | Y        | 链表结构有界阻塞队列         |
| LinkedBlockingDeque   | 实现类 | Y    | Y        | 链表结构双向有界阻塞队列     |
| LinkedTransferQueue   | 实现类 | N    | Y        | 连接结构无界阻塞传输队列     |
| SynchronousQueue      | 实现类 | Y    | Y        | 不存储元素的有界阻塞队列     |
| PriorityBlockingQueue | 实现类 | N    | Y        | 支持优先级排序的无界阻塞队列 |
| DelayQueue            | 实现类 | N    | Y        | 延时无界阻塞队列             |

## 层级结构

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430134832.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

## 1. Queue

- Queue接口是一种Collection，被设计用于处理之前临时保存在某处的元素。
- 除了基本的Collection操作之外，队列还提供了额外的插入、提取和检查操作。每一种操作都有两种形式：如果操作失败，则抛出一个异常；如果操作失败，则返回一个特殊值（null或false，取决于是什么操作）。
- 队列通常是以FIFO（先进先出）的方式排序元素，但是这不是必须的。
- 只有优先级队列可以根据提供的比较器对元素进行排序或者是采用正常的排序。无论怎么排序，队列的头将通过调用remove()或poll()方法进行移除。在FIFO队列种，所有新的元素被插入到队尾。其他种类的队列可能使用不同的布局来存放元素。
- 每个Queue必须指定排序属性。

## 2. Deque

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430135110.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

支持两端元素插入和移除的线性集合。名称`deque`是双端队列的缩写(Double-Ended queue)，通常发音为`deck`。大多数实现Deque的类，对它们包含的元素的数量没有固定的限制的，支持有界和无界。

- 该列表包含包含访问deque两端元素的方法，提供了插入，移除和检查元素的方法。
- 这些方法种的每一种都存在两种形式：如果操作失败，则会抛出异常，另一种方法返回一个特殊值（null或false，取决于具体操作）。
- 插入操作的后一种形式专门设计用于容量限制的Deque实现，大多数实现中，插入操作不能失败，所以可以用插入操作的后一种形式。
- Deque接口扩展了Queue接口，当使用deque作为队列时，作为FIFO。元素将添加到deque的末尾，并从头开始删除。
- Deque也可以用作LIFO（后进先出）栈，这个接口优于传统的Stack类。当作为栈使用时，元素被push到deque队列的头，而pop也是从队列的头pop出来。

## 3. AbstractQueue

AbstractQueue是一个抽象类，继承了Queue接口，提供了一些Queue操作的骨架实现。

方法add、remove、element方法基于offer、poll和peek。也就是说如果不能正常操作，则抛出异常。我们来看下AbstactQueue是怎么做到的。

- AbstractQueue的add方法

```java
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```

- AbstractQueue的remove方法

```java
public E remove() {
    E x = poll();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
```

- AbstractQueue的element方法

```java
public E element() {
    E x = peek();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
```

如果继承AbstractQueue抽象类则必须保证offer方法不允许null值插入。

## 4. BlockingQueue

- BlockQueue满了，PUT操作被阻塞

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430135520.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- BlockQueue为空，Take操作被阻塞

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430135535.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

说明：

- BlockingQueue（阻塞队列）也是一种队列，支持阻塞的插入和移除方法。
- 阻塞的插入：当队列满时，队列会阻塞插入元素的线程，直到队列不满。
- 阻塞的移除：当队列为空，获取元素的线程会等待队列变为非空。
- 应用场景：生产者和消费者，生产者线程向队列里添加元素，消费者线程从队列里移除元素，阻塞队列时获取和存放元素的容器。
- 为什么要用阻塞队列：生产者生产和消费者消费的速率不一样，需要用队列来解决速率差问题，当队列满了或空的时候，则需要阻塞生产或消费动作来解决队列满或空的问题。

方法总结：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430135640.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### 如何实现的阻塞

- 当往队列里插入一个元素时，如果队列不可用，那么阻塞生产者主要通过LockSupport. park（this）来实现。
- park这个方法会阻塞当前线程，只有以下4种情况中的一种发生时，该方法才会返回。
  - 与park对应的unpark执行或已经执行时。“已经执行”是指unpark先执行，然后再执行park的情况。
  - 线程被中断时。
  - 等待完time参数指定的毫秒数时。
  - 异常现象发生时，这个异常现象没有任何原因。

## 5. BlockingDeque

- BlockingDeque 满了，两端的 put 操作被阻塞

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430135836.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- BlockingDeque 为空，两端的Take操作被阻塞

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430135912.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

它是阻塞队列`BlockingQueue`和双向队列`Deque`接口的结合。有如下方法：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430135939.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

BlockDeque和BlockQueue的对等方法：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430140007.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

## 6. TransferQueue

如果有消费者正在获取元素，则将队列中的元素传递给消费者。如果没有消费者，则等待消费者消费。必须将任务完成才能返回。

#### transfer(E e)

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430140158.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- 生产者线程Producer Thread尝试将元素B传给消费者线程，如果没有消费者线程，则将元素B放到尾节点。并且生产者线程等待元素B被消费。当元素B被消费后，生产者线程返回。
- 如果当前有消费者正在等待接收元素（消费者通过take方法或超时限制的poll方法时），transfer方法可以把生产者传入的元素立刻transfer（传输）给消费者。
- 如果没有消费者等待接收元素，transfer方法会将元素放在队列的tail（尾）节点，并等到该元素被消费者消费了才返回。

#### tryTransfer(E e)

- 试探生产者传入的元素是否能直接传给消费者。
- 如果没有消费者等待接收元素，则返回false。
- 和transfer方法的区别是，无论消费者是否接收，方法立即返回。

#### tryTransfer(E e, long timeout, TimeUnit unit)

- 带有时间限制的tryTransfer方法。
- 试图把生产者传入的元素直接传给消费者。
- 如果没有消费者消费该元素则等待指定的时间再返回。
- 如果超时了还没有消费元素，则返回false。
- 如果在超时时间内消费了元素，则返回true。

#### getWaitingConsumerCount()

- 获取通过BlockingQueue.take()方法或超时限制poll方法等待接受元素的消费者数量。近似值。
- 返回等待接收元素的消费者数量。

#### hasWaitingConsumer()

- 获取是否有通过BlockingQueue.tabke()方法或超时限制poll方法等待接受元素的消费者。
- 返回true则表示至少有一个等待消费者。

## 7. PriorityQueue

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430140515.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430140507.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- PriorityQueue是一个支持优先级的无界阻塞队列。
- **默认自然顺序升序排序**。
- 可以通过构造参数Comparator来对元素进行排序。
- 自定义实现comapreTo()方法来指定元素排序规则。
- 不允许插入null元素。
- 实现PriorityQueue接口的类，不保证线程安全，除非是PriorityBlockingQueue。
- PriorityQueue的迭代器不能保证以任何特定顺序遍历元素，如果需要有序遍历，请考虑使用 `Arrays.sort(pq.toArray)`。
- 进列( `offer`、 `add`)和出列（ `poll`、 `remove()`）的时间复杂度O(log(n))。
- remove(Object) 和 contains(Object)的算法时间复杂度O(n)。
- peek、element、size的算法时间复杂度为O(1)。

## 8. LinkedList

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430140626.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- LinkedList实现了List和Deque接口，所以是一种双链表结构，可以当作堆栈、队列、双向队列使用。
- 一个双向列表的每一个元素都有三个整数值：元素、向后的节点链接、向前的节点链接

```java
private static class Node<E> {
    E item; //元素
    Node<E> next; //向后的节点链接
    Node<E> prev; //向前的节点链接

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

## 9. ConcurrentLinkedQueue

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430140725.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- ConcurrentLinked是由链表结构组成的线程安全的先进先出无界队列。
- 当多线程要共享访问集合时，ConcurrentLinkedQueue是一个比较好的选择。
- 不允许插入null元素
- 支持非阻塞地访问并发安全的队列，不会抛出ConcurrentModifiationException异常。
- size方法不是准确的，因为在统计集合的时候，队列可能正在添加元素，导致统计不准。
- 批量操作addAll、removeAll、retainAll、containsAll、equals和toArray不保证原子性（操作不可分割）
- 添加元素happen-before其他线程移除元素。

## 10. ArrayDeque

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430141318.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- 由数组组成的双端队列。
- 没有容量限制，根据需要扩容。
- 不是线程安全的。
- 禁止插入null元素。
- 当用作栈时，比栈速度快，当用作队列时，速度比LinkList快。
- 大部分方法的算法时间复杂度为O(1)。
- remove、removeFirstOccurrence、removeLastOccurrence、contains、remove 和批量操作的算法时间复杂度O(n)

## 11. ConcurrentLinkedDeque

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430141422.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- 由链表结构组成的双向无界阻塞队列
- 插入、删除和访问操作可以并发进行，线程安全的类
- 不允许插入null元素
- 在并发场景下，计算队列的大小是不准确的，因为计算时，可能有元素加入队列。
- 批量操作addAll、removeAll、retainAll、containsAll、equals和toArray不保证原子性（操作不可分割）

## 12. ArrayBlockingQueue

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430141457.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- ArrayBlockingQueue是一个用数组实现的有界阻塞队列。
- 队列满时插入操作被阻塞，队列空时，移除操作被阻塞。
- 按照先进先出（FIFO）原则对元素进行排序。
- 默认不保证线程公平的访问队列。
- 公平访问队列：按照阻塞的先后顺序访问队列，即先阻塞的线程先访问队列。
- 非公平性是对先等待的线程是非公平的，当队列可用时，阻塞的线程都可以争夺访问队列的资格。有可能先阻塞的线程最后才访问访问队列。
- 公平性会降低吞吐量。

## 13. LinkedBlockinQueue

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430141557.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- LinkedBlockingQueue具有单链表和有界阻塞队列的功能。
- 队列满时插入操作被阻塞，队列空时，移除操作被阻塞。
- 默认和最大长度为Integer.MAX_VALUE，相当于无界(值非常大：2^31-1)。
- 吞吐量通常要高于ArrayBlockingQueue。
- 创建线程池时，参数runnableTaskQueue（任务队列），用于保存等待执行的任务的阻塞队列可以选择LinkedBlockingQueue。
- 静态工厂方法Executors.newFixedThreadPool()使用了这个队列。

## 14. LinkedBlockingDeque

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430141703.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- 由链LinkedBlockingDeque = 阻塞队列+链表+双端访问
- 线程安全。
- 多线程同时入队时，因多了一端访问入口，所以减少了一半的竞争。
- 默认容量大小为Integer.MAX_VALUE。可指定容量大小。
- 可以用在“工作窃取“模式中。

## 15. LinkedTransferQueue

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430141750.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

LinkedTransferQueue = 阻塞队列+链表结构+TransferQueue

之前我们讲TransferQueue接口时**已经介绍过了TransferQueue接口** ，所以LinkedTransferQueue接口跟它相似，只是加入了阻塞插入和移除的功能，以及结构是链表结构。

## 16. SynchronousQueue

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430141858.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- 我称SynchronousQueue为”传球好手“。想象一下这个场景：小明抱着一个篮球想传给小花，如果小花没有将球拿走，则小明是不能再拿其他球的。
- SynchronousQueue负责把生产者产生的数据传递给消费者线程。
- SynchronousQueue本身不存储数据，调用了put方法后，队列里面也是空的。
- 每一个put操作必须等待一个take操作完成，否则不能添加元素。
- 适合传递性场景。
- 性能高于ArrayBlockingQueue和LinkedBlockingQueue。
- 吞吐量通常要高于LinkedBlockingQueue。
- 创建线程池时，参数runnableTaskQueue（任务队列），用于保存等待执行的任务的阻塞队列可以选择SynchronousQueue。
- 静态工厂方法Executors.newCachedThreadPool()使用了这个队列

## 17. PriorityBlockQueue

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430142029.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- PriorityBlockQueue = PriorityQueue + BlockingQueue
- 之前我们也讲到了PriorityQueue的原理，支持对元素排序。
- 元素默认自然升序排序。
- 可以自定义CompareTo()方法来指定元素排序规则。
- 可以通过构造函数构造参数Comparator来对元素进行排序。

## 18. DelayQueue

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430142532.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- DelayQueue = Delayed + BlockingQueue。队列中的元素必须实现Delayed接口。
- 在创建元素时，可以指定多久可以从队列中获取到当前元素。只有在延时期满才能从队列中获取到当前元素。

场景：

- 缓存系统的设计：可以用DelayQueue保存缓存元素的有效期。然后用一个线程循环的查询DelayQueue队列，一旦能从DelayQueue中获取元素时，表示缓存有效期到了。
- 定时任务调度：使用DelayQueue队列保存当天将会执行的任务和执行时间，一旦从DelayQueue中获取到任务就开始执行。比如Java中的TimerQueue就是使用DelayQueue实现的。

