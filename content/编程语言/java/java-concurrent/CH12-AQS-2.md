---
type: docs
title: "CH12-AQS-2"
linkTitle: "CH12-AQS-2"
weight: 12
---

## 应用示例

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class MyThread extends Thread {
    private Lock lock;
    public MyThread(String name, Lock lock) {
        super(name);
        this.lock = lock;
    }
    
    public void run () {
        lock.lock();
        try {
            System.out.println(Thread.currentThread() + " running");
        } finally {
            lock.unlock();
        }
    }
}
public class AbstractQueuedSynchonizerDemo {
    public static void main(String[] args) {
        Lock lock = new ReentrantLock();
        
        MyThread t1 = new MyThread("t1", lock);
        MyThread t2 = new MyThread("t2", lock);
        t1.start();
        t2.start();    
    }
}

// 前后随机
Thread[t1,5,main] running
Thread[t2,5,main] running
```

从示例可知，线程t1与t2共用了一把锁，即同一个lock。可能会存在如下一种时序：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210426235454.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

首先 t1 线程调用 lock.lock 操作，然后 t2 再执行 lock.lock 操作，然后 t1 执行 lock.unlock，最后 t2 执行 lock.unlock。基于这样的时序尝试分析 AQS 内部的机制。

- t1 线程调用 lock.lock 操作：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210426235655.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- t2 再执行 lock.lock 操作：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210426235730.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

进过一系列方法调用，最后达到的状态是 t2 被禁用，因此调用了 LockSupport.lock。

- t1线程调用lock.unlock：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210426235830.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

t1线程中调用lock.unlock后，经过一系列的调用，最终的状态是释放了许可，因为调用了LockSupport.unpark。这时，t2线程就可以继续运行了。此时，会继续恢复t2线程运行环境，继续执行LockSupport.park后面的语句，即进一步调用如下。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210426235856.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

在上一步调用了LockSupport.unpark后，t2线程恢复运行，则运行parkAndCheckInterrupt，之后，继续运行acquireQueued方法，最后达到的状态是头结点head与尾结点tail均指向了t2线程所在的结点，并且之前的头结点已经从sync队列中断开了。

- t2线程调用lock.unlock，其方法调用顺序如下，只给出了主要的方法调用。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210426235925.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

t2线程执行lock.unlock后，最终达到的状态还是与之前的状态一样。