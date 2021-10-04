---
type: docs
title: "CH19-BlockingQueue"
linkTitle: "CH19-BlockingQueue"
weight: 19
---

## BlockingQueue

通常用于一个线程生产对象，而另外一个线程消费这些对象的场景。下图是对这个原理的阐述:

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427232026.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

一个线程将会持续生产新对象并将其插入到队列之中，直到队列达到它所能容纳的临界点。也就是说，它是有限的。如果该阻塞队列到达了其临界点，负责生产的线程将会在往里边插入新对象时发生阻塞。它会一直处于阻塞之中，直到负责消费的线程从队列中拿走一个对象。 负责消费的线程将会一直从该阻塞队列中拿出对象。如果消费线程尝试去从一个空的队列中提取对象的话，这个消费线程将会处于阻塞之中，直到一个生产线程把一个对象丢进队列。

### 操作方法

具有 4 组不同的方法用于插入、移除以及对队列中的元素进行检查。如果请求的操作不能得到立即执行的话，每个方法的表现也不同。这些方法如下:

|      | 抛异常     | 布尔值   | 阻塞    | 超时                      |
| ---- | ---------- | -------- | ------- | ------------------------- |
| 插入 | add(o)     | offer(o) | put(o)  | offer(o,timeout,timeunit) |
| 移除 | remove(o)  | poll(o)  | take(o) | poll(timeout,timeunit)    |
| 检查 | element(o) | peek(o)  |         |                           |

- 抛异常：如果试图的操作无法立即执行，抛一个异常。
- 特定值：如果试图的操作无法立即执行，返回一个特定的值(常常是 true / false)。
- 阻塞：如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行。
- 超时：如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行，但等待时间不会超过给定值。返回一个特定值以告知该操作是否成功(典型的是 true / false)。

无法向一个 BlockingQueue 中插入 null。如果你试图插入 null，BlockingQueue 将会抛出一个 NullPointerException。

可以访问到 BlockingQueue 中的所有元素，而不仅仅是开始和结束的元素。比如说，你将一个对象放入队列之中以等待处理，但你的应用想要将其取消掉。那么你可以调用诸如 remove(o) 方法来将队列之中的特定对象进行移除。但是这么干效率并不高(译者注: 基于队列的数据结构，获取除开始或结束位置的其他对象的效率不会太高)，因此你尽量不要用这一类的方法，除非你确实不得不那么做。

## BlockingDeque

BlockingDeque 接口表示一个线程安放入和提取实例的双端队列。

BlockingDeque 类是一个双端队列，在不能够插入元素时，它将阻塞住试图插入元素的线程；在不能够抽取元素时，它将阻塞住试图抽取的线程。 deque(双端队列) 是 "Double Ended Queue" 的缩写。因此，双端队列是一个你可以从任意一端插入或者抽取元素的队列。

在线程既是一个队列的生产者又是这个队列的消费者的时候可以使用到 BlockingDeque。如果生产者线程需要在队列的两端都可以插入数据，消费者线程需要在队列的两端都可以移除数据，这个时候也可以使用 BlockingDeque。BlockingDeque 图解:

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427232657.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### 操作方法

一个 BlockingDeque - 线程在双端队列的两端都可以插入和提取元素。 一个线程生产元素，并把它们插入到队列的任意一端。如果双端队列已满，插入线程将被阻塞，直到一个移除线程从该队列中移出了一个元素。如果双端队列为空，移除线程将被阻塞，直到一个插入线程向该队列插入了一个新元素。

BlockingDeque 具有 4 组不同的方法用于插入、移除以及对双端队列中的元素进行检查。如果请求的操作不能得到立即执行的话，每个方法的表现也不同。这些方法如下:

|           | 抛异常         | 布尔值        | 阻塞         | 超时                             |
| --------- | -------------- | ------------- | ------------ | -------------------------------- |
| 队首-插入 | addFirst(o)    | offerFirst(o) | putFirst(o)  | offerFirst(o, timeout, timeunit) |
| 队首-移除 | removeFirst(o) | pollFirst(o)  | takeFirst(o) | pollFirst(timeout, timeunit)     |
| 队首-检查 | getFirst(o)    | peekFirst(o)  |              |                                  |
| 队尾-插入 | addLast(o)     | offerLast(o)  | putLast(o)   | offerLast(o, timeout, timeunit)  |
| 队尾-移除 | removeLast(o)  | pollLast(o)   | takeLast(o)  | pollLast(timeout, timeunit)      |
| 队尾-检查 | getLast(o)     | peekLast(o)   |              |                                  |

- 抛异常：如果试图的操作无法立即执行，抛一个异常。
- 特定值：如果试图的操作无法立即执行，返回一个特定的值(常常是 true / false)。
- 阻塞：如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行。
- 超时：如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行，但等待时间不会超过给定值。返回一个特定值以告知该操作是否成功(典型的是 true / false)。

## BlockingQueue & BlockingDeque

BlockingDeque 接口继承自 BlockingQueue 接口。这就意味着你可以像使用一个 BlockingQueue 那样使用 BlockingDeque。如果你这么干的话，各种插入方法将会把新元素添加到双端队列的尾端，而移除方法将会把双端队列的首端的元素移除。正如 BlockingQueue 接口的插入和移除方法一样。

## 应用实例

这里是一个 Java 中使用 BlockingQueue 的示例。本示例使用的是 BlockingQueue 接口的 ArrayBlockingQueue 实现。 首先，BlockingQueueExample 类分别在两个独立的线程中启动了一个 Producer 和 一个 Consumer。Producer 向一个共享的 BlockingQueue 中注入字符串，而 Consumer 则会从中把它们拿出来。

```java
public class BlockingQueueExample {
 
    public static void main(String[] args) throws Exception {
 
        BlockingQueue queue = new ArrayBlockingQueue(1024);
 
        Producer producer = new Producer(queue);
        Consumer consumer = new Consumer(queue);
 
        new Thread(producer).start();
        new Thread(consumer).start();
 
        Thread.sleep(4000);
    }
}
```

以下是 Producer 类。注意它在每次 put() 调用时是如何休眠一秒钟的。这将导致 Consumer 在等待队列中对象的时候发生阻塞。

```java
public class Producer implements Runnable{
 
    protected BlockingQueue queue = null;
 
    public Producer(BlockingQueue queue) {
        this.queue = queue;
    }
 
    public void run() {
        try {
            queue.put("1");
            Thread.sleep(1000);
            queue.put("2");
            Thread.sleep(1000);
            queue.put("3");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

以下是 Consumer 类。它只是把对象从队列中抽取出来，然后将它们打印到 System.out。

```java
public class Consumer implements Runnable{
 
    protected BlockingQueue queue = null;
 
    public Consumer(BlockingQueue queue) {
        this.queue = queue;
    }
 
    public void run() {
        try {
            System.out.println(queue.take());
            System.out.println(queue.take());
            System.out.println(queue.take());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## ArrayBlockingQueue

ArrayBlockingQueue 类实现了 BlockingQueue 接口。

ArrayBlockingQueue 是一个有界的阻塞队列，其内部实现是将对象放到一个数组里。有界也就意味着，它不能够存储无限多数量的元素。它有一个同一时间能够存储元素数量的上限。你可以在对其初始化的时候设定这个上限，但之后就无法对这个上限进行修改了。 

ArrayBlockingQueue 内部以 FIFO(先进先出)的顺序对元素进行存储。队列中的头元素在所有元素之中是放入时间最久的那个，而尾元素则是最短的那个。

## DelayQueue

DelayQueue 实现了 BlockingQueue 接口。

DelayQueue是一个无界的 BlockingQueue，用于放置实现了 Delayed 接口的对象，其中的对象只能在其到期时才能从队列中取走。这种队列是有序的，即队头对象的延迟到期时间最长。

元素进入队列后，先进行排序，然后，只有 getDelay 也就是剩余时间为0的时候，该元素才有资格被消费者从队列中取出来，所以构造函数一般都有一个时间传入。

```java
public interface Delayed extends Comparable<Delayed< {
    public long getDelay(TimeUnit timeUnit);
}
```

传递给 getDelay 方法的 getDelay 实例是一个枚举类型，它表明了将要延迟的时间段。

Delayed 接口也继承了 java.lang.Comparable 接口，这也就意味着 Delayed 对象之间可以进行对比。这个可能在对 DelayQueue 队列中的元素进行排序时有用，因此它们可以根据过期时间进行有序释放。 以下是使用 DelayQueue 的例子:

```java
public class DelayQueueExample {
 
    public static void main(String[] args) {
        DelayQueue queue = new DelayQueue();
        Delayed element1 = new DelayedElement();
        queue.put(element1);
        Delayed element2 = queue.take();
    }
}
```

## LinkedBlocingQueue

LinkedBlockingQueue 类实现了 BlockingQueue 接口。

LinkedBlockingQueue 内部以一个链式结构(链接节点)对其元素进行存储。如果需要的话，这一链式结构可以选择一个上限。如果没有定义上限，将使用 Integer.MAX_VALUE 作为上限。

LinkedBlockingQueue 内部以 FIFO(先进先出)的顺序对元素进行存储。队列中的头元素在所有元素之中是放入时间最久的那个，而尾元素则是最短的那个。 

## PriorityBlockingQueue

PriorityBlockingQueue 类实现了 BlockingQueue 接口。

PriorityBlockingQueue 是一个无界的并发队列。它使用了和类 java.util.PriorityQueue 一样的排序规则。你无法向这个队列中插入 null 值。 所有插入到 PriorityBlockingQueue 的元素必须实现 java.lang.Comparable 接口。因此该队列中元素的排序就取决于你自己的 Comparable 实现。 注意 PriorityBlockingQueue 对于具有相等优先级(compare() == 0)的元素并不强制任何特定行为。

同时注意，如果你从一个 PriorityBlockingQueue 获得一个 Iterator 的话，该 Iterator 并不能保证它对元素的遍历是以优先级为序的。

## SynchronousQueue

SynchronousQueue 类实现了 BlockingQueue 接口。

SynchronousQueue 是一个特殊的队列，它的内部同时只能够容纳单个元素。如果该队列已有一元素的话，试图向队列中插入一个新元素的线程将会阻塞，直到另一个线程将该元素从队列中抽走。同样，如果该队列为空，试图向队列中抽取一个元素的线程将会阻塞，直到另一个线程向队列中插入了一条新的元素。 据此，把这个类称作一个队列显然是夸大其词了。它更多像是一个汇合点。

