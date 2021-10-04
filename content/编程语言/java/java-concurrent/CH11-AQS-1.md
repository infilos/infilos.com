---
type: docs
title: "CH11-AQS-1"
linkTitle: "CH11-AQS-1"
weight: 11
---

## AbstractQueuedSynchronizer

AQS 是一个用来构建锁和同步器的框架，使用 AQS 能够简单高效的构造出应用广泛的同步器，比如 ReentrantLock、Semaphore，其他诸如 ReentrantReadWriteLock、SynchronousQueue、FutureTask 等也是基于 AQS 实现的。我们自己也可以基于 AQS 构造满足自己需要的同步器。

### 核心思想

- 如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。
- 如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制。
  - AQS 使用 CLH 队列锁实现了该机制，将暂时获取不到锁的线程加入到队列中。
  - AQS 使用一个 int 成员变量表示同步状态，通过内置的 FIFO 队列来完成获取资源线程的排队工作。
  - AQS 使用 CAS 对该同步状态执行原子操作以实现值的修改，并使用 volatile 保证该状态的可见性。

> CLH(Craig,Landin,and Hagersten) 队列是一个虚拟的双向队列(即不存在队列实例、仅存在结点之间的关联关系)。AQS是将每条请求共享资源的线程封装成一个 CLH 锁队列的一个结点(Node)来实现锁的分配。

状态信息通过 protected 范围的方法执行操作：

```java
private volatile int state;

//返回同步状态的当前值
protected final int getState() {  
        return state;
}
 // 设置同步状态的值
protected final void setState(int newState) { 
        state = newState;
}
//原子地(CAS操作)将同步状态值设置为给定值update如果当前同步状态的值等于expect(期望值)
protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

### 资源共享方式

- 独占(Exclusive)：只有一个线程能够执行，如 ReentrantLock。又可以分为公平锁和非公平锁：
  - 公平锁：按照线程在队列中的排队顺序，先到者先获得锁。
  - 非公平锁：当线程要获得锁时，无视队列顺序直接抢锁，谁抢到随获取。
- 共享(Share)：多个线程可以同时执行，如 Semaphore、CountDownLatch。

而 ReentrantReadWriteLock 可以看做是对以上两种方式的组合，因为它允许多个线程同时对一个资源执行读，但仅能有一个线程执行写。

不同的自定义同步器争用共享资源的方式不同。自定义同步器在实现时只需要实现共享资源 state 的获取与释放方式即可，至于具体线程等待队列的维护(如获取资源失败后入队/唤醒出队等)，AQS 已经在上层实现了。

### AQS 底层使用的模板方法模式

同步器的设计基于模板方法模式，自定义同步器时继承 AQS 并重写指定的方法即可：

- isHeldExclusively：判断线程是否正在独占资源，只有用到 condition 才需要实现。
- tryAcquire(int)：独占获取资源，成功失败返回 ture、false。
- tryRelease(int)：独占释放资源，成功失败返回 true、false。
- tryAcquireShared(int)：共享获取资源，失败为负，为 0 表示成功但没有剩余可用资源，为正表示成且有可用资源。
- tryReleaseShared(int)：共享释放资源，成功失败返回 true、false。

相关细节：

- 默认情况下，每个方法都能抛出 UnsupportedException。
- 所有方法的实现必须是内部线程安全的，并且应该简短且不阻塞。
- 其他方法均为 final，所以仅有以上方法可以被其他类使用。

以 ReentrantLock 为例：

- 初始状态 state 为 0，表示未锁定状态。
- A 线程 lock 时，会调用 tryAcquire 独占锁定并将 state+1。
- 此后其他线程再 tryAcquire 时会失败，直到 A 线程 unlock 并将 state=0(释放锁)为止。
- 在 A 线程释放锁之前，A 线程可以重复获取此锁(state 累加)，即可重入。
  - 获取多次就要释放多次，直至 state 为 0 才表示释放锁。

## 数据结构

- AQS 底层使用 CLH，将每条请求共享资源的线程封装为 CLH 队列的一个节点。
- 其中同步队列 Sync Queue 为双向链表，包括 head 和 tail 节点，head 节点主要用作后续的调度。
- 其中 Condition Queue 不是必须，是一个单向链表，只有使用 Condition 时，才会使用该队列。
  - 并且可能会有多个 Condition Queue。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/java-thread-x-juc-aqs-1.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

## 源码分析

### 层级结构

AQS 继承抽象类 AbstractOwnableSynchronizer，实现了 Serializable 接口，支持序列化。

```java
public abstract class AbstractOwnableSynchronizer implements java.io.Serializable {
    
    // 版本序列号
    private static final long serialVersionUID = 3737899427754241961L;
    // 构造方法
    protected AbstractOwnableSynchronizer() { }
    // 独占模式下的线程
    private transient Thread exclusiveOwnerThread;
    
    // 设置独占线程 
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
    
    // 获取独占线程 
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

- 其中可以“设置独占资源线程”和“获取独占资源线程”，分别为 setExclusiveOwnerThread 与 getExclusiveOwnerThread 方法，这两个方法会被子类调用。
- 其中有两个内部类，Node、ConditionObject。

### 内部类：Node

```java
static final class Node {
    // 模式，分为共享与独占
    // 共享模式
    static final Node SHARED = new Node();
    // 独占模式
    static final Node EXCLUSIVE = null;        
    // 结点状态
    // CANCELLED，值为1，表示当前的线程被取消
    // SIGNAL，值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark
    // CONDITION，值为-2，表示当前节点在等待condition，也就是在condition队列中
    // PROPAGATE，值为-3，表示当前场景下后续的acquireShared能够得以执行
    // 值为0，表示当前节点在sync队列中，等待着获取锁
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;        

    // 结点状态
    volatile int waitStatus;        
    // 前驱结点
    volatile Node prev;    
    // 后继结点
    volatile Node next;        
    // 结点所对应的线程
    volatile Thread thread;        
    // 下一个等待者
    Node nextWaiter;
    
    // 结点是否在共享模式下等待
    final boolean isShared() {
        return nextWaiter == SHARED;
    }
    
    // 获取前驱结点，若前驱结点为空，抛出异常
    final Node predecessor() throws NullPointerException {
        // 保存前驱结点
        Node p = prev; 
        if (p == null) // 前驱结点为空，抛出异常
            throw new NullPointerException();
        else // 前驱结点不为空，返回
            return p;
    }
    
    // 无参构造方法
    Node() {    // Used to establish initial head or SHARED marker
    }
    
    // 构造方法
        Node(Thread thread, Node mode) {    // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }
    
    // 构造方法
    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

- 每个被阻塞的线程都会被封装为一个 Node 节点并放入队列。
- 每个节点包含了一个 Thread 类型的引用，并且每个节点都有一个状态，状态如下：
  - CANCELLED：1，当前线程被取消。
  - SIGNAL：-1，当前节点的后继节点中的线程需要运行，需要进行 unpark 操作。
  - CONDITION：-2，当前节点在等待 condition，即 condition queue 中。
  - PROPAGATE：-3，当前场景下后续的 acquireSHared 能够得以执行。
  - 0：当前节点在 sync queue 中，等待获取锁。

### 内部类：ConditionObject

```java
// 内部类
public class ConditionObject implements Condition, java.io.Serializable {
    // 版本号
    private static final long serialVersionUID = 1173984872572414699L;
    /** First node of condition queue. */
    // condition队列的头结点
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    // condition队列的尾结点
    private transient Node lastWaiter;

    /**
        * Creates a new {@code ConditionObject} instance.
        */
    // 构造方法
    public ConditionObject() { }

    // Internal methods

    /**
        * Adds a new waiter to wait queue.
        * @return its new wait node
        */
    // 添加新的waiter到wait队列
    private Node addConditionWaiter() {
        // 保存尾结点
        Node t = lastWaiter;
        // If lastWaiter is cancelled, clean out.
        if (t != null && t.waitStatus != Node.CONDITION) { // 尾结点不为空，并且尾结点的状态不为CONDITION
            // 清除状态为CONDITION的结点
            unlinkCancelledWaiters(); 
            // 将最后一个结点重新赋值给t
            t = lastWaiter;
        }
        // 新建一个结点
        Node node = new Node(Thread.currentThread(), Node.CONDITION);
        if (t == null) // 尾结点为空
            // 设置condition队列的头结点
            firstWaiter = node;
        else // 尾结点不为空
            // 设置为节点的nextWaiter域为node结点
            t.nextWaiter = node;
        // 更新condition队列的尾结点
        lastWaiter = node;
        return node;
    }

    /**
        * Removes and transfers nodes until hit non-cancelled one or
        * null. Split out from signal in part to encourage compilers
        * to inline the case of no waiters.
        * @param first (non-null) the first node on condition queue
        */
    private void doSignal(Node first) {
        // 循环
        do {
            if ( (firstWaiter = first.nextWaiter) == null) // 该节点的nextWaiter为空
                // 设置尾结点为空
                lastWaiter = null;
            // 设置first结点的nextWaiter域
            first.nextWaiter = null;
        } while (!transferForSignal(first) &&
                    (first = firstWaiter) != null); // 将结点从condition队列转移到sync队列失败并且condition队列中的头结点不为空，一直循环
    }

    /**
        * Removes and transfers all nodes.
        * @param first (non-null) the first node on condition queue
        */
    private void doSignalAll(Node first) {
        // condition队列的头结点尾结点都设置为空
        lastWaiter = firstWaiter = null;
        // 循环
        do {
            // 获取first结点的nextWaiter域结点
            Node next = first.nextWaiter;
            // 设置first结点的nextWaiter域为空
            first.nextWaiter = null;
            // 将first结点从condition队列转移到sync队列
            transferForSignal(first);
            // 重新设置first
            first = next;
        } while (first != null);
    }

    /**
        * Unlinks cancelled waiter nodes from condition queue.
        * Called only while holding lock. This is called when
        * cancellation occurred during condition wait, and upon
        * insertion of a new waiter when lastWaiter is seen to have
        * been cancelled. This method is needed to avoid garbage
        * retention in the absence of signals. So even though it may
        * require a full traversal, it comes into play only when
        * timeouts or cancellations occur in the absence of
        * signals. It traverses all nodes rather than stopping at a
        * particular target to unlink all pointers to garbage nodes
        * without requiring many re-traversals during cancellation
        * storms.
        */
    // 从condition队列中清除状态为CANCEL的结点
    private void unlinkCancelledWaiters() {
        // 保存condition队列头结点
        Node t = firstWaiter;
        Node trail = null;
        while (t != null) { // t不为空
            // 下一个结点
            Node next = t.nextWaiter;
            if (t.waitStatus != Node.CONDITION) { // t结点的状态不为CONDTION状态
                // 设置t节点的额nextWaiter域为空
                t.nextWaiter = null;
                if (trail == null) // trail为空
                    // 重新设置condition队列的头结点
                    firstWaiter = next;
                else // trail不为空
                    // 设置trail结点的nextWaiter域为next结点
                    trail.nextWaiter = next;
                if (next == null) // next结点为空
                    // 设置condition队列的尾结点
                    lastWaiter = trail;
            }
            else // t结点的状态为CONDTION状态
                // 设置trail结点
                trail = t;
            // 设置t结点
            t = next;
        }
    }

    // public methods

    /**
        * Moves the longest-waiting thread, if one exists, from the
        * wait queue for this condition to the wait queue for the
        * owning lock.
        *
        * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
        *         returns {@code false}
        */
    // 唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从 await 返回之前，该线程必须重新获取锁。
    public final void signal() {
        if (!isHeldExclusively()) // 不被当前线程独占，抛出异常
            throw new IllegalMonitorStateException();
        // 保存condition队列头结点
        Node first = firstWaiter;
        if (first != null) // 头结点不为空
            // 唤醒一个等待线程
            doSignal(first);
    }

    /**
        * Moves all threads from the wait queue for this condition to
        * the wait queue for the owning lock.
        *
        * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
        *         returns {@code false}
        */
    // 唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程。在从 await 返回之前，每个线程都必须重新获取锁。
    public final void signalAll() {
        if (!isHeldExclusively()) // 不被当前线程独占，抛出异常
            throw new IllegalMonitorStateException();
        // 保存condition队列头结点
        Node first = firstWaiter;
        if (first != null) // 头结点不为空
            // 唤醒所有等待线程
            doSignalAll(first);
    }

    /**
        * Implements uninterruptible condition wait.
        * <ol>
        * <li> Save lock state returned by {@link #getState}.
        * <li> Invoke {@link #release} with saved state as argument,
        *      throwing IllegalMonitorStateException if it fails.
        * <li> Block until signalled.
        * <li> Reacquire by invoking specialized version of
        *      {@link #acquire} with saved state as argument.
        * </ol>
        */
    // 等待，当前线程在接到信号之前一直处于等待状态，不响应中断
    public final void awaitUninterruptibly() {
        // 添加一个结点到等待队列
        Node node = addConditionWaiter();
        // 获取释放的状态
        int savedState = fullyRelease(node);
        boolean interrupted = false;
        while (!isOnSyncQueue(node)) { // 
            // 阻塞当前线程
            LockSupport.park(this);
            if (Thread.interrupted()) // 当前线程被中断
                // 设置interrupted状态
                interrupted = true; 
        }
        if (acquireQueued(node, savedState) || interrupted) // 
            selfInterrupt();
    }

    /*
        * For interruptible waits, we need to track whether to throw
        * InterruptedException, if interrupted while blocked on
        * condition, versus reinterrupt current thread, if
        * interrupted while blocked waiting to re-acquire.
        */

    /** Mode meaning to reinterrupt on exit from wait */
    private static final int REINTERRUPT =  1;
    /** Mode meaning to throw InterruptedException on exit from wait */
    private static final int THROW_IE    = -1;

    /**
        * Checks for interrupt, returning THROW_IE if interrupted
        * before signalled, REINTERRUPT if after signalled, or
        * 0 if not interrupted.
        */
    private int checkInterruptWhileWaiting(Node node) {
        return Thread.interrupted() ?
            (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
            0; 
    }

    /**
        * Throws InterruptedException, reinterrupts current thread, or
        * does nothing, depending on mode.
        */
    private void reportInterruptAfterWait(int interruptMode)
        throws InterruptedException {
        if (interruptMode == THROW_IE)
            throw new InterruptedException();
        else if (interruptMode == REINTERRUPT)
            selfInterrupt();
    }

    /**
        * Implements interruptible condition wait.
        * <ol>
        * <li> If current thread is interrupted, throw InterruptedException.
        * <li> Save lock state returned by {@link #getState}.
        * <li> Invoke {@link #release} with saved state as argument,
        *      throwing IllegalMonitorStateException if it fails.
        * <li> Block until signalled or interrupted.
        * <li> Reacquire by invoking specialized version of
        *      {@link #acquire} with saved state as argument.
        * <li> If interrupted while blocked in step 4, throw InterruptedException.
        * </ol>
        */
    // // 等待，当前线程在接到信号或被中断之前一直处于等待状态
    public final void await() throws InterruptedException {
        if (Thread.interrupted()) // 当前线程被中断，抛出异常
            throw new InterruptedException();
        // 在wait队列上添加一个结点
        Node node = addConditionWaiter();
        // 
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            // 阻塞当前线程
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) // 检查结点等待时的中断类型
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }

    /**
        * Implements timed condition wait.
        * <ol>
        * <li> If current thread is interrupted, throw InterruptedException.
        * <li> Save lock state returned by {@link #getState}.
        * <li> Invoke {@link #release} with saved state as argument,
        *      throwing IllegalMonitorStateException if it fails.
        * <li> Block until signalled, interrupted, or timed out.
        * <li> Reacquire by invoking specialized version of
        *      {@link #acquire} with saved state as argument.
        * <li> If interrupted while blocked in step 4, throw InterruptedException.
        * </ol>
        */
    // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态 
    public final long awaitNanos(long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        final long deadline = System.nanoTime() + nanosTimeout;
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            if (nanosTimeout <= 0L) {
                transferAfterCancelledWait(node);
                break;
            }
            if (nanosTimeout >= spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
            nanosTimeout = deadline - System.nanoTime();
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
        return deadline - System.nanoTime();
    }

    /**
        * Implements absolute timed condition wait.
        * <ol>
        * <li> If current thread is interrupted, throw InterruptedException.
        * <li> Save lock state returned by {@link #getState}.
        * <li> Invoke {@link #release} with saved state as argument,
        *      throwing IllegalMonitorStateException if it fails.
        * <li> Block until signalled, interrupted, or timed out.
        * <li> Reacquire by invoking specialized version of
        *      {@link #acquire} with saved state as argument.
        * <li> If interrupted while blocked in step 4, throw InterruptedException.
        * <li> If timed out while blocked in step 4, return false, else true.
        * </ol>
        */
    // 等待，当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态
    public final boolean awaitUntil(Date deadline)
            throws InterruptedException {
        long abstime = deadline.getTime();
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        boolean timedout = false;
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            if (System.currentTimeMillis() > abstime) {
                timedout = transferAfterCancelledWait(node);
                break;
            }
            LockSupport.parkUntil(this, abstime);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
        return !timedout;
    }

    /**
        * Implements timed condition wait.
        * <ol>
        * <li> If current thread is interrupted, throw InterruptedException.
        * <li> Save lock state returned by {@link #getState}.
        * <li> Invoke {@link #release} with saved state as argument,
        *      throwing IllegalMonitorStateException if it fails.
        * <li> Block until signalled, interrupted, or timed out.
        * <li> Reacquire by invoking specialized version of
        *      {@link #acquire} with saved state as argument.
        * <li> If interrupted while blocked in step 4, throw InterruptedException.
        * <li> If timed out while blocked in step 4, return false, else true.
        * </ol>
        */
    // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。此方法在行为上等效于: awaitNanos(unit.toNanos(time)) > 0
    public final boolean await(long time, TimeUnit unit)
            throws InterruptedException {
        long nanosTimeout = unit.toNanos(time);
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        final long deadline = System.nanoTime() + nanosTimeout;
        boolean timedout = false;
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            if (nanosTimeout <= 0L) {
                timedout = transferAfterCancelledWait(node);
                break;
            }
            if (nanosTimeout >= spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
            nanosTimeout = deadline - System.nanoTime();
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
        return !timedout;
    }

    //  support for instrumentation

    /**
        * Returns true if this condition was created by the given
        * synchronization object.
        *
        * @return {@code true} if owned
        */
    final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
        return sync == AbstractQueuedSynchronizer.this;
    }

    /**
        * Queries whether any threads are waiting on this condition.
        * Implements {@link AbstractQueuedSynchronizer#hasWaiters(ConditionObject)}.
        *
        * @return {@code true} if there are any waiting threads
        * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
        *         returns {@code false}
        */
    //  查询是否有正在等待此条件的任何线程
    protected final boolean hasWaiters() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
            if (w.waitStatus == Node.CONDITION)
                return true;
        }
        return false;
    }

    /**
        * Returns an estimate of the number of threads waiting on
        * this condition.
        * Implements {@link AbstractQueuedSynchronizer#getWaitQueueLength(ConditionObject)}.
        *
        * @return the estimated number of waiting threads
        * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
        *         returns {@code false}
        */
    // 返回正在等待此条件的线程数估计值
    protected final int getWaitQueueLength() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        int n = 0;
        for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
            if (w.waitStatus == Node.CONDITION)
                ++n;
        }
        return n;
    }

    /**
        * Returns a collection containing those threads that may be
        * waiting on this Condition.
        * Implements {@link AbstractQueuedSynchronizer#getWaitingThreads(ConditionObject)}.
        *
        * @return the collection of threads
        * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
        *         returns {@code false}
        */
    // 返回包含那些可能正在等待此条件的线程集合
    protected final Collection<Thread> getWaitingThreads() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
            if (w.waitStatus == Node.CONDITION) {
                Thread t = w.thread;
                if (t != null)
                    list.add(t);
            }
        }
        return list;
    }
}
```

该类实现了 Condition 接口，Condition 接口定义了条件操作的规范：

```java
public interface Condition {

    // 等待，当前线程在接到信号或被中断之前一直处于等待状态
    void await() throws InterruptedException;
    
    // 等待，当前线程在接到信号之前一直处于等待状态，不响应中断
    void awaitUninterruptibly();
    
    //等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态 
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    
    // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。此方法在行为上等效于: awaitNanos(unit.toNanos(time)) > 0
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    
    // 等待，当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态
    boolean awaitUntil(Date deadline) throws InterruptedException;
    
    // 唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从 await 返回之前，该线程必须重新获取锁。
    void signal();
    
    // 唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程。在从 await 返回之前，每个线程都必须重新获取锁。
    void signalAll();
}
```

### 类的属性

属性中包含了头结点 head，为节点 tail，状态 state，自旋时间 spinForTimeoutThreshold，以及 AQS 抽象的属性在内存中的便宜地址，通过该便宜地址，可以获取和设置属性的值，同时该包括一个静态初始化块，用于加载内存偏移地址：

```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer
    implements java.io.Serializable {    
    // 版本号
    private static final long serialVersionUID = 7373984972572414691L;    
    // 头结点
    private transient volatile Node head;    
    // 尾结点
    private transient volatile Node tail;    
    // 状态
    private volatile int state;    
    // 自旋时间
    static final long spinForTimeoutThreshold = 1000L;
    
    // Unsafe类实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // state内存偏移地址
    private static final long stateOffset;
    // head内存偏移地址
    private static final long headOffset;
    // state内存偏移地址
    private static final long tailOffset;
    // tail内存偏移地址
    private static final long waitStatusOffset;
    // next内存偏移地址
    private static final long nextOffset;
    // 静态初始化块
    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("next"));

        } catch (Exception ex) { throw new Error(ex); }
    }
}
```

### 类的构造方法

该类构造方法为抽象构造方法，仅供子类调用。

### 类的核心方法：acquire

该方法以独占模式获取(资源)，忽略中断，即线程在aquire过程中，中断此线程是无效的。源码如下:

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

线程在调用 tryAcquire 时的流程如下：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210426234229.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- 首先调用 tryAcquire 方法，线程会尝试在独占模式下获取对象状态。
  - 此方法会查询是否允许它在独占模式下获取对象状态，如果允许则获取。
  - 在 AQS 源码中会默认抛出一个异常，即需要子类重写该方法以实现需要的逻辑。
- 若 tryAcquire 失败，则调用 addWaiter 方法，addWaiter 方法完成的功能是将调用此方法的线程封装成为一个节点并放入 Sync Queue。
- 调用 acquireQueued 方法，此方法完成的功能是 Sync Queue 中的节点不断尝试获取资源，成功失败返回 true、false。
- 调用 tryAcquire 默认实现是抛出异常，因此需要继承者实现。

首先是 addWaiter 方法：

```java
// 添加等待者
private Node addWaiter(Node mode) {
    // 新生成一个结点，默认为独占模式
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 保存尾结点
    Node pred = tail;
    if (pred != null) { // 尾结点不为空，即已经被初始化
        // 将node结点的prev域连接到尾结点
        node.prev = pred; 
        if (compareAndSetTail(pred, node)) { // 比较pred是否为尾结点，是则将尾结点设置为node 
            // 设置尾结点的next域为node
            pred.next = node;
            return node; // 返回新生成的结点
        }
    }
    enq(node); // 尾结点为空(即还没有被初始化过)，或者是compareAndSetTail操作失败，则入队列
    return node;
}
```

addWaiter 方法使用快速添加的方式往 sync queue 尾部添加结点，如果 sync queue 队列还没有初始化，则会使用 enq 插入队列中，enq 方法源码如下：

```java
private Node enq(final Node node) {
    for (;;) { // 无限循环，确保结点能够成功入队列
        // 保存尾结点
        Node t = tail;
        if (t == null) { // 尾结点为空，即还没被初始化
            if (compareAndSetHead(new Node())) // 头结点为空，并设置头结点为新生成的结点
                tail = head; // 头结点与尾结点都指向同一个新生结点
        } else { // 尾结点不为空，即已经被初始化过
            // 将node结点的prev域连接到尾结点
            node.prev = t; 
            if (compareAndSetTail(t, node)) { // 比较结点t是否为尾结点，若是则将尾结点设置为node
                // 设置尾结点的next域为node
                t.next = node; 
                return t; // 返回尾结点
            }
        }
    }
}
```

enq 方法会使用无限循环来确保节点的成功插入。

acquireQueue方法：

```java
// sync队列中的结点在独占且忽略中断的模式下获取(资源)
final boolean acquireQueued(final Node node, int arg) {
    // 标志
    boolean failed = true;
    try {
        // 中断标志
        boolean interrupted = false;
        for (;;) { // 无限循环
            // 获取node节点的前驱结点
            final Node p = node.predecessor(); 
            if (p == head && tryAcquire(arg)) { // 前驱为头结点并且成功获得锁
                setHead(node); // 设置头结点
                p.next = null; // help GC
                failed = false; // 设置标志
                return interrupted; 
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

首先获取当前节点的前驱节点，如果前驱节点是头结点并且能够获取(资源)，代表该当前节点能够占有锁，设置头结点为当前节点，返回。否则，调用shouldParkAfterFailedAcquire和parkAndCheckInterrupt方法，首先，我们看shouldParkAfterFailedAcquire方法，代码如下：

```java
// 当获取(资源)失败后，检查并且更新结点状态
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 获取前驱结点的状态
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL) // 状态为SIGNAL，为-1
        /*
            * This node has already set status asking a release
            * to signal it, so it can safely park.
            */
        // 可以进行park操作
        return true; 
    if (ws > 0) { // 表示状态为CANCELLED，为1
        /*
            * Predecessor was cancelled. Skip over predecessors and
            * indicate retry.
            */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0); // 找到pred结点前面最近的一个状态不为CANCELLED的结点
        // 赋值pred结点的next域
        pred.next = node; 
    } else { // 为PROPAGATE -3 或者是0 表示无状态,(为CONDITION -2时，表示此节点在condition queue中) 
        /*
            * waitStatus must be 0 or PROPAGATE.  Indicate that we
            * need a signal, but don't park yet.  Caller will need to
            * retry to make sure it cannot acquire before parking.
            */
        // 比较并设置前驱结点的状态为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL); 
    }
    // 不能进行park操作
    return false;
}
```

只有当该节点的前驱结点的状态为SIGNAL时，才可以对该结点所封装的线程进行park操作。否则，将不能进行park操作。再看parkAndCheckInterrupt方法，源码如下：

```java
// 进行park操作并且返回该线程是否被中断
private final boolean parkAndCheckInterrupt() {
    // 在许可可用之前禁用当前线程，并且设置了blocker
    LockSupport.park(this);
    return Thread.interrupted(); // 当前线程是否已被中断，并清除中断标记位
}
```

parkAndCheckInterrupt方法里的逻辑是首先执行park操作，即禁用当前线程，然后返回该线程是否已经被中断。再看final块中的cancelAcquire方法，其源码如下：

```java
// 取消继续获取(资源)
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    // node为空，返回
    if (node == null)
        return;
    // 设置node结点的thread为空
    node.thread = null;

    // Skip cancelled predecessors
    // 保存node的前驱结点
    Node pred = node.prev;
    while (pred.waitStatus > 0) // 找到node前驱结点中第一个状态小于0的结点，即不为CANCELLED状态的结点
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    // 获取pred结点的下一个结点
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    // 设置node结点的状态为CANCELLED
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) { // node结点为尾结点，则设置尾结点为pred结点
        // 比较并设置pred结点的next节点为null
        compareAndSetNext(pred, predNext, null); 
    } else { // node结点不为尾结点，或者比较设置不成功
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
                (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) { // (pred结点不为头结点，并且pred结点的状态为SIGNAL)或者 
                                // pred结点状态小于等于0，并且比较并设置等待状态为SIGNAL成功，并且pred结点所封装的线程不为空
            // 保存结点的后继
            Node next = node.next;
            if (next != null && next.waitStatus <= 0) // 后继不为空并且后继的状态小于等于0
                compareAndSetNext(pred, predNext, next); // 比较并设置pred.next = next;
        } else {
            unparkSuccessor(node); // 释放node的前一个结点
        }

        node.next = node; // help GC
    }
}
```

该方法完成的功能就是取消当前线程对资源的获取，即设置该结点的状态为CANCELLED，接着我们再看unparkSuccessor方法，源码如下：

```java
// 释放后继结点
private void unparkSuccessor(Node node) {
    /*
        * If status is negative (i.e., possibly needing signal) try
        * to clear in anticipation of signalling.  It is OK if this
        * fails or if status is changed by waiting thread.
        */
    // 获取node结点的等待状态
    int ws = node.waitStatus;
    if (ws < 0) // 状态值小于0，为SIGNAL -1 或 CONDITION -2 或 PROPAGATE -3
        // 比较并且设置结点等待状态，设置为0
        compareAndSetWaitStatus(node, ws, 0);

    /*
        * Thread to unpark is held in successor, which is normally
        * just the next node.  But if cancelled or apparently null,
        * traverse backwards from tail to find the actual
        * non-cancelled successor.
        */
    // 获取node节点的下一个结点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) { // 下一个结点为空或者下一个节点的等待状态大于0，即为CANCELLED
        // s赋值为空
        s = null; 
        // 从尾结点开始从后往前开始遍历
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0) // 找到等待状态小于等于0的结点，找到最前的状态小于等于0的结点
                // 保存结点
                s = t;
    }
    if (s != null) // 该结点不为为空，释放许可
        LockSupport.unpark(s.thread);
}
```

该方法的作用就是为了释放node节点的后继结点。

对于cancelAcquire与unparkSuccessor方法，如下示意图可以清晰的表示:

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210426235112.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

其中node为参数，在执行完cancelAcquire方法后的效果就是unpark了s结点所包含的t4线程。

现在，再来看acquireQueued方法的整个的逻辑。逻辑如下:

1. 判断结点的前驱是否为head并且是否成功获取(资源)。

2. 若步骤1均满足，则设置结点为head，之后会判断是否finally模块，然后返回。

3. 若步骤2不满足，则判断是否需要park当前线程，是否需要park当前线程的逻辑是判断结点的前驱结点的状态是否为SIGNAL，若是，则park当前结点，否则，不进行park操作。
4. 若park了当前线程，之后某个线程对本线程unpark后，并且本线程也获得机会运行。那么，将会继续进行步骤 1 的判断。

### 类的核心方法：release

以独占模式释放资源：

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) { // 释放成功
        // 保存头结点
        Node h = head; 
        if (h != null && h.waitStatus != 0) // 头结点不为空并且头结点状态不为0
            unparkSuccessor(h); //释放头结点的后继结点
        return true;
    }
    return false;
}
```

其中，tryRelease的默认实现是抛出异常，需要具体的子类实现，如果tryRelease成功，那么如果头结点不为空并且头结点的状态不为0，则释放头结点的后继结点，unparkSuccessor方法已经分析过，不再累赘。

对于其他方法我们也可以分析，与前面分析的方法大同小异，所以，不再累赘。

## 参考资料

- [AQS-CLH 锁](https://www.infoq.cn/article/BVPvyVxjKM8ZSTSpTi0L)