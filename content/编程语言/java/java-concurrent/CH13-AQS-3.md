---
type: docs
title: "CH13-AQS-3"
linkTitle: "CH13-AQS-3"
weight: 13
---

## 应用实例

下面我们结合Condition实现生产者与消费者，来进一步分析AbstractQueuedSynchronizer的内部工作机制。

Depot(仓库)类：

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Depot {
    private int size;
    private int capacity;
    private Lock lock;
    private Condition fullCondition;
    private Condition emptyCondition;
    
    public Depot(int capacity) {
        this.capacity = capacity;    
        lock = new ReentrantLock();
        fullCondition = lock.newCondition();
        emptyCondition = lock.newCondition();
    }
    
    public void produce(int no) {
        lock.lock();
        int left = no;
        try {
            while (left > 0) {
                while (size >= capacity)  {
                    System.out.println(Thread.currentThread() + " before await");
                    fullCondition.await();
                    System.out.println(Thread.currentThread() + " after await");
                }
                int inc = (left + size) > capacity ? (capacity - size) : left;
                left -= inc;
                size += inc;
                System.out.println("produce = " + inc + ", size = " + size);
                emptyCondition.signal();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
    
    public void consume(int no) {
        lock.lock();
        int left = no;
        try {            
            while (left > 0) {
                while (size <= 0) {
                    System.out.println(Thread.currentThread() + " before await");
                    emptyCondition.await();
                    System.out.println(Thread.currentThread() + " after await");
                }
                int dec = (size - left) > 0 ? left : size;
                left -= dec;
                size -= dec;
                System.out.println("consume = " + dec + ", size = " + size);
                fullCondition.signal();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

测试类：

```java
class Consumer {
    private Depot depot;
    public Consumer(Depot depot) {
        this.depot = depot;
    }
    
    public void consume(int no) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                depot.consume(no);
            }
        }, no + " consume thread").start();
    }
}

class Producer {
    private Depot depot;
    public Producer(Depot depot) {
        this.depot = depot;
    }
    
    public void produce(int no) {
        new Thread(new Runnable() {
            
            @Override
            public void run() {
                depot.produce(no);
            }
        }, no + " produce thread").start();
    }
}

public class ReentrantLockDemo {
    public static void main(String[] args) throws InterruptedException {
        Depot depot = new Depot(500);
        new Producer(depot).produce(500);
        new Producer(depot).produce(200);
        new Consumer(depot).consume(500);
        new Consumer(depot).consume(200);
    }
}
```

运行结果(随机)：

```bash
produce = 500, size = 500
Thread[200 produce thread,5,main] before await
consume = 500, size = 0
Thread[200 consume thread,5,main] before await
Thread[200 produce thread,5,main] after await
produce = 200, size = 200
Thread[200 consume thread,5,main] after await
consume = 200, size = 0
```

根据结果，我们猜测一种可能的时序如下：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000103.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

p1代表produce 500的那个线程，p2代表produce 200的那个线程，c1代表consume 500的那个线程，c2代表consume 200的那个线程。

- p1线程调用lock.lock，获得锁，继续运行，方法调用顺序在前面已经给出。
- p2线程调用lock.lock，由前面的分析可得到如下的最终状态。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000133.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

p2线程调用lock.lock后，会禁止p2线程的继续运行，因为执行了LockSupport.park操作。

- c1线程调用lock.lock，由前面的分析得到如下的最终状态。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000153.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

最终c1线程会在sync queue队列的尾部，并且其结点的前驱结点(包含p2的结点)的waitStatus变为了SIGNAL。

- c2线程调用lock.lock，由前面的分析得到如下的最终状态。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000215.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

最终c1线程会在sync queue队列的尾部，并且其结点的前驱结点(包含c1的结点)的waitStatus变为了SIGNAL。

- p1线程执行emptyCondition.signal，其方法调用顺序如下，只给出了主要的方法调用。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000235.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

AQS.CO表示AbstractQueuedSynchronizer.ConditionObject类。此时调用signal方法不会产生任何其他效果。

- p1线程执行lock.unlock，根据前面的分析可知，最终的状态如下。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000259.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

此时，p2线程所在的结点为头结点，并且其他两个线程(c1、c2)依旧被禁止，所以，此时p2线程继续运行，执行用户逻辑。

- p2线程执行fullCondition.await，其方法调用顺序如下，只给出了主要的方法调用。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000318.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

最终到达的状态是新生成了一个结点，包含了p2线程，此结点在condition queue中；并且sync queue中p2线程被禁止了，因为在执行了LockSupport.park操作。从方法一些调用可知，在await操作中线程会释放锁资源，供其他线程获取。同时，head结点后继结点的包含的线程的许可被释放了，故其可以继续运行。由于此时，只有c1线程可以运行，故运行c1。

- 继续运行c1线程，c1线程由于之前被park了，所以此时恢复，继续之前的步骤，即还是执行前面提到的acquireQueued方法，之后，c1判断自己的前驱结点为head，并且可以获取锁资源，最终到达的状态如下。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000409.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

其中，head设置为包含c1线程的结点，c1继续运行。

- c1线程执行fullCondtion.signal，其方法调用顺序如下，只给出了主要的方法调用。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000431.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

signal方法达到的最终结果是将包含p2线程的结点从condition queue中转移到sync queue中，之后condition queue为null，之前的尾结点的状态变为SIGNAL。

- c1线程执行lock.unlock操作，根据之前的分析，经历的状态变化如下。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000459.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

最终c2线程会获取锁资源，继续运行用户逻辑。

- c2线程执行emptyCondition.await，由前面的第七步分析，可知最终的状态如下。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000518.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

await操作将会生成一个结点放入condition queue中与之前的一个condition queue是不相同的，并且unpark头结点后面的结点，即包含线程p2的结点。

- p2线程被unpark，故可以继续运行，经过CPU调度后，p2继续运行，之后p2线程在AQS:await方法中被park，继续AQS.CO:await方法的运行，其方法调用顺序如下，只给出了主要的方法调用。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000544.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- p2继续运行，执行emptyCondition.signal，根据第九步分析可知，最终到达的状态如下。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000619.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

最终，将condition queue中的结点转移到sync queue中，并添加至尾部，condition queue会为空，并且将head的状态设置为SIGNAL。

- p2线程执行lock.unlock操作，根据前面的分析可知，最后的到达的状态如下。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210427000652.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

unlock操作会释放c2线程的许可，并且将头结点设置为c2线程所在的结点。

- c2线程继续运行，执行fullCondition. signal，由于此时fullCondition的condition queue已经不存在任何结点了，故其不会产生作用。
- c2执行lock.unlock，由于c2是sync队列中最后一个结点，故其不会再调用unparkSuccessor了，直接返回true。即整个流程就完成了。