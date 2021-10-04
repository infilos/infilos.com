---
type: docs
title: "CH30-AllLocks"
linkTitle: "CH30-AllLocks"
weight: 30
---

## 概览

| 序号 | 术语     | 应用                                                         |
| ---- | -------- | ------------------------------------------------------------ |
| 1    | 乐观锁   | CAS                                                          |
| 2    | 悲观锁   | synchronized、vector、hashtable                              |
| 3    | 自旋锁   | CAS                                                          |
| 4    | 可重入锁 | synchronized、ReentrantLock、Lock                            |
| 5    | 读写锁   | ReentrantReadWriteLock、CopyOnWriteLock、CopyOnWriteArraySet |
| 6    | 公平锁   | ReentrantLock(true)                                          |
| 7    | 非公平锁 | synchronized、ReentrantLock(false)                           |
| 8    | 共享锁   | ReentranReadWriteLock-ReadLock                               |
| 9    | 独占锁   | synchronized、vector、hashtable、ReentranReadWriteLock-WriteLock |
| 10   | 重量级锁 | synchronized                                                 |
| 11   | 轻量级锁 | 锁优化技术                                                   |
| 12   | 偏向锁   | 锁优化技术                                                   |
| 13   | 分段锁   | ConcurrentHashMap                                            |
| 14   | 互斥锁   | synchronized                                                 |
| 15   | 同步锁   | synchronized                                                 |
| 16   | 死锁     | 相互请求对方资源                                             |
| 17   | 锁粗化   | 锁优化技术                                                   |
| 18   | 锁消除   | 锁优化技术                                                   |

## 1. 乐观锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429204833.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

即乐观思想，假定当前场景是读多写少、遇到并发写的概览较低，读数据时认为别的线程不会正在修改数据(因此不加锁)；写数据时，判断当前与期望值是否相同，如果相同则更新(更新期间加锁，保证原子性)。

Java 中乐观锁的实现是 CAS——比较并交换。比较(主内存中的)当前值，与(当前线程中的)预期值是否一样，一样则更新，否则继续进行 CAS 操作。

可以同时进行读操作，读的时候其他线程不能执行写操作。

## 2. 悲观锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429205158.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

即悲观思想，认为写多读少，遇到并发写的可能性高。每次读数据都认为其他线程会在同一时间修改数据，所以每次写数据都会认为其他线程会修改，因此每次都加锁。其他线程想要读写这个数据时都会被该锁阻塞，直到当前写数据的线程是否锁。

Java 中的悲观锁实现有 synchronized 关键字、ReentrantLock。

只有一个线程能够进行读操作或写操作。

## 3. 自旋锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429205437.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

自旋指的是一种行为：为了让线程等待，我们只需让该线程循环。

现在绝大多数的个人电脑和服务器都是多路（核）处理器系统，如果物理机器有一个以上的处理器或者处理器核心，能让两个或以上的线程同时并行执行，就可以让后面请求锁的那个线程“稍等一会”，但不放弃处理器的执行时间，看看持有锁的线程是否很快就会释放锁。

优点：避免了线程切换的开销。挂起线程和恢复线程的操作都需要转入内核态中完成，这些操作给Java虚拟机的并发性能带来了很大的压力。

缺点：占用处理器的时间，如果占用的时间很长，会白白消耗处理器资源，而不会做任何有价值的工作，带来性能的浪费。因此自旋等待的时间必须有一定的限度，如果自旋超过了限定的次数仍然没有成功获得锁，就应当使用传统的方式去挂起线程。

Java 中默认的自旋次数为 10，可以通过参数 `-XX:PreBlockSpin` 来修改。

自适应自旋：自适应意味着自旋的时间不再是固定的，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定的。有了自适应自旋，随着程序运行时间的增长及性能监控信息的不断完善，虚拟机对程序锁的状态预测就会越来越精准。

Java 中对自旋的应用：CAS 操作中比较操作失败后会执行自旋等待。

## 4. 可重入锁(递归锁)

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429205752.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

可重入指的是：某个线程在获取到锁之后能够再次获取该锁，而不会阻塞。

原理：通过组合自定义同步器来实现锁的获取与释放。

- 再次获取锁：识别获取锁的线程是否为当前持有锁的线程，如果是则再次获取成功，并将技术 +1。
- 释放锁：释放锁并将计数 -1。

作用：避免死锁。

Java 中的实现有：ReentrantLock、synchronized 关键字。

## 5. 读写锁

读写锁指定是指：为了提高性能，在读的时候使用读锁，写的时候使用写锁，灵活控制。在没有写的时候，读是无阻塞的，在一定程度上提高了程序的执行效率。

读写锁分为读锁和写锁，多个读锁不互斥，读锁与写锁互斥。

读锁：允许多个线程同时访问资源。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429210319.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

写锁：同时只允许一个线程访问资源。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429210345.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

Java 中的实现为 ReentrantReadWriteLock。

## 6. 公平锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429210418.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

公平锁的思想是：多个线程按照请求所的顺序来依次获取锁。

在并发环境中，每个线程会先查看此锁维护的等待队列，如果当前等待队列为空，则占有锁，如果等待队列不为空，则加入到等待队列的末尾，按照FIFO的原则从队列中拿到线程，然后占有锁。

## 7. 非公平锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429210515.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

非公平锁的思想是：线程尝试获取锁，如果获取不到，则再采用公平锁的方式。多个线程获取锁的顺序，不是按照先到先得的顺序，有可能后申请锁的线程比先申请的线程优先获取锁。

非公平锁的性能高于公平锁，但可能导致某个线程总是获取不到锁，即饥饿。

Java 中的实现：synchronized 是非公平锁，ReentrantLock 通过构造函数指定该锁是公平的还是非公平的，默认是非公平的。

## 8. 共享锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429210709.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

共享锁的思想是：可以有多个线程获取读锁，以共享的方式持有锁。和乐观锁、读写锁同义。

**Java中用到的共享锁：** `ReentrantReadWriteLock`。

## 9. 独占锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429210751.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

独占锁的思想是：只能有一个线程获取锁，以独占的方式持有锁。和悲观锁、互斥锁同义。

**Java中用到的独占锁：** synchronized，ReentrantLock

## 10. 重量级锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429210822.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**重量级锁是一种称谓：** `synchronized`是通过对象内部的一个叫做监视器锁（`monitor`）来实现的，监视器锁本身依赖底层的操作系统的 `Mutex Lock`来实现。操作系统实现线程的切换需要从用户态切换到核心态，成本非常高。这种依赖于操作系统 `Mutex Lock`来实现的锁称为重量级锁。为了优化`synchonized`，引入了`轻量级锁`，`偏向锁`。

**Java中的重量级锁：** synchronized

## 11. 轻量级锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429210904.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**是JDK6时加入的一种锁优化机制：** 轻量级锁是在无竞争的情况下使用CAS操作去消除同步使用的互斥量。轻量级是相对于使用操作系统互斥量来实现的重量级锁而言的。轻量级锁在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。如果出现两条以上的线程争用同一个锁的情况，那轻量级锁将不会有效，必须膨胀为重量级锁。

**优点：** 如果没有竞争，通过CAS操作成功避免了使用互斥量的开销。

**缺点：** 如果存在竞争，除了互斥量本身的开销外，还额外产生了CAS操作的开销，因此在有竞争的情况下，轻量级锁比传统的重量级锁更慢。

## 12. 偏向锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429210945.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**是JDK6时加入的一种锁优化机制：** 在无竞争的情况下把整个同步都消除掉，连CAS操作都不去做了。偏是指偏心，它的意思是这个锁会偏向于第一个获得它的线程，如果在接下来的执行过程中，该锁一直没有被其他的线程获取，则持有偏向锁的线程将永远不需要再进行同步。持有偏向锁的线程以后每次进入这个锁相关的同步块时，虚拟机都可以不再进行任何同步操作（例如加锁、解锁及对Mark Word的更新操作等）。

**优点：** 把整个同步都消除掉，连CAS操作都不去做了，优于轻量级锁。

**缺点：** 如果程序中大多数的锁都总是被多个不同的线程访问，那偏向锁就是多余的。

## 13. 分段锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429211027.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**一种机制：** 最好的例子来说明分段锁是ConcurrentHashMap。**ConcurrentHashMap原理：**它内部细分了若干个小的 HashMap，称之为段(Segment)。默认情况下一个 ConcurrentHashMap 被进一步细分为 16 个段，既就是锁的并发度。如果需要在 ConcurrentHashMap 添加一项key-value，并不是将整个 HashMap 加锁，而是首先根据 hashcode 得到该key-value应该存放在哪个段中，然后对该段加锁，并完成 put 操作。在多线程环境中，如果多个线程同时进行put操作，只要被加入的key-value不存放在同一个段中，则线程间可以做到真正的并行。

**线程安全：**ConcurrentHashMap 是一个 Segment 数组， Segment 通过继承ReentrantLock 来进行加锁，所以每次需要加锁的操作锁住的是一个 segment，这样只要保证每个 Segment 是线程安全的，也就实现了全局的线程安全

## 14. 互斥锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429211130.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

互斥锁与悲观锁、独占锁同义，表示某个资源只能被一个线程访问，其他线程不能访问。

- 读-读互斥
- 读-写互斥
- 写-读互斥
- 写-写互斥

**Java中的同步锁：** synchronized

## 15. 同步锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429211204.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

同步锁与互斥锁同义，表示并发执行的多个线程，在同一时间内只允许一个线程访问共享数据。

**Java中的同步锁：** synchronized

## 16. 死锁

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429211225.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**死锁是一种现象：**如线程A持有资源x，线程B持有资源y，线程A等待线程B释放资源y，线程B等待线程A释放资源x，两个线程都不释放自己持有的资源，则两个线程都获取不到对方的资源，就会造成死锁。

Java中的死锁不能自行打破，所以线程死锁后，线程不能进行响应。所以一定要注意程序的并发场景，避免造成死锁。

## 17. 锁粗化

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429211257.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**一种优化技术：** 如果一系列的连续操作都对同一个对象反复加锁和解锁，甚至加锁操作都是出现在循环体体之中，就算真的没有线程竞争，频繁地进行互斥同步操作将会导致不必要的性能损耗，所以就采取了一种方案：把加锁的范围扩展（粗化）到整个操作序列的外部，这样加锁解锁的频率就会大大降低，从而减少了性能损耗。

## 18. 锁消除

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210429211343.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**一种优化技术：** 就是把锁干掉。当Java虚拟机运行时发现有些共享数据不会被线程竞争时就可以进行锁消除。

那如何判断共享数据不会被线程竞争？

利用`逃逸分析技术`：分析对象的作用域，如果对象在A方法中定义后，被作为参数传递到B方法中，则称为方法逃逸；如果被其他线程访问，则称为线程逃逸。

在堆上的某个数据不会逃逸出去被其他线程访问到，就可以把它当作栈上数据对待，认为它是线程私有的，同步加锁就不需要了。

## TODO

- https://tech.meituan.com/2018/11/15/java-lock.html