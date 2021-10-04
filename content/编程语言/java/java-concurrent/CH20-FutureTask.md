---
type: docs
title: "CH20-FutureTask"
linkTitle: "CH20-FutureTask"
weight: 20
---

## 概览

- FutureTask 为 Future 提供了基础实现，如获取任务执行结果(get)和取消任务等。
- 如果任务尚未完成，获取任务执行结果时将会阻塞。
- 一旦执行结束，任务就不能被重启或取消(除非使用runAndReset执行计算)。
- FutureTask 常用来封装 Callable 和 Runnable，也可以作为一个任务提交到线程池中执行。
- 除了作为一个独立的类之外，此类也提供了一些功能性函数供我们创建自定义 task 类使用。
- FutureTask 的线程安全由 CAS 来保证。

## 层级结构

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210428222223.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

FutureTask 实现了 RunnableFuture 接口，则 RunnableFuture 接口继承了 Runnable 接口和 Future 接口，所以 FutureTask 既能当做一个 Runnable 直接被 Thread 执行，也能作为 Future 用来得到 Callable 的计算结果。

## 源码分析

### Callable 接口

Callable 是个泛型接口，泛型V就是要 call() 方法返回的类型。对比 Runnable 接口，Runnable 不会返回数据也不能抛出异常。

```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

### Future 接口

Future 接口代表异步计算的结果，通过 Future 接口提供的方法可以查看异步计算是否执行完成，或者等待执行结果并获取执行结果，同时还可以取消执行。Future 接口的定义如下:

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

- cancel：取消异步任务的执行。
  - 如果异步任务已经完成或者已经被取消，或者由于某些原因不能取消，则会返回 false。
  - 如果任务还没有被执行，则会返回 true 并且异步任务不会被执行。
  - 如果任务已经开始执行了但是还没有执行完成：
    - 若 mayInterruptIfRunning 为 true，则会立即中断执行任务的线程并返回 true；
    - 若 mayInterruptIfRunning 为 false，则会返回 true 且不会中断任务执行线程。
- isCanceled：判断任务是否被取消。
  - 如果任务在结束(正常执行结束或者执行异常结束)前被取消则返回 true，
  - 否则返回 false。
- isDone：判断任务是否已经完成。
  - 如果完成则返回 true，否则返回 false。
  - 任务执行过程中发生异常、任务被取消也属于任务已完成，也会返回true。
- get：获取任务执行结果。
  - 如果任务还没完成则会阻塞等待直到任务执行完成。
  - 如果任务被取消则会抛出 CancellationException 异常。
  - 如果任务执行过程发生异常则会抛出 ExecutionException 异常。
  - 如果阻塞等待过程中被中断则会抛出 InterruptedException 异常。
- get(timeout,timeunit)：带超时时间的 get() 版本。
  - 如果阻塞等待过程中超时则会抛出 TimeoutException 异常。

### 核心属性

```java

//内部持有的callable任务，运行完毕后置空
private Callable<V> callable;

//从get()中返回的结果或抛出的异常
private Object outcome; // non-volatile, protected by state reads/writes

//运行callable的线程
private volatile Thread runner;

//使用Treiber栈保存等待线程
private volatile WaitNode waiters;

//任务状态
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

其中的状态值 state 使用 volatile 修饰，以确保任何一个线程对状态的修改立即会对其他线程可见。

7 种具体状态表示：

- NEW：初始状态，表示这个是新任务或者尚未被执行完的任务。
- COMPLETING：任务已经执行完成或者执行任务的时候发生异常。
  - 但是任务执行结果或者异常原因还没有保存到 outcome 字段时，状态由 NEW 变为 COMPLETING。
  - outcome字段用来保存任务执行结果，如果发生异常，则用来保存异常原因。
  - 该状态持续时间较短，属于中间状态。
- NORMAL：任务已经执行完成并且任务执行结果已经保存到 outcome 字段，状态会从 COMPLETING 转换到 NORMAL。
  - 这是一个最终态。
- EXCEPTIONAL：任务执行发生异常并且异常原因已经保存到 outcome 字段中后，状态会从 COMPLETING 转换到 EXCEPTIONAL。
  - 这是一个最终态。
- CANCELED：任务还没开始执行或者已经开始执行但是还没有执行完成的时候，用户调用了 `cancel(false)` 方法取消任务且不中断任务执行线程，这个时候状态会从 NEW 转化为 CANCELLED 状态。
  - 这是一个最终态。
- INTERRUPTING：任务还没开始执行或者已经执行但是还没有执行完成的时候，用户调用了 `cancel(true)` 方法取消任务并且要中断任务执行线程但是还没有中断任务执行线程之前，状态会从 NEW 转化为 INTERRUPTING。
  - 这是一个中间状态。
- INTERRUPTED：调用 interrupt() 中断任务执行线程之后状态会从 INTERRUPTING 转换到 INTERRUPTED。
  - 这是一个最终态。 
  - 所有值大于 COMPLETING 的状态都表示任务已经执行完成(任务正常执行完成，任务执行异常或者任务被取消)。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210428223530.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### 构造函数

#### `FutureTask(Callable<V> callable)`

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
```

- 该构造函数会把传入的 Callable 变量保存在t his.callable 字段中。
- 该字段定义为`private Callable<V> callable`; 用来保存底层的调用，在被执行完成以后会指向 null。
- 接着会初始化 state 字段为 NEW。

#### `FutureTask(Runnable runnable, V result)`

```java
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

- 这个构造函数会把传入的 Runnable 封装成一个 Callable 对象保存在 callable 字段中。
- 同时如果任务执行成功的话就会返回传入的 result。
- 如果不需要返回值的话可以传入一个 null 作为 result。
- Executors.callable() 的功能是把 Runnable 转换成 Callable。

```java
public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
       throw new NullPointerException();
    return new RunnableAdapter<T>(task, result); // 适配器
}
```

这里采用了适配器模式：

```java
static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
    public T call() {
        task.run();
        return result;
    }
}
```

这里的适配器只是简单实现了 Callable 接口，在 call 中调用 Runnable.run 方法，然后把传入的 result 作为返回值返回调用。

在 new 了一个 FutureTask 之后，接下来就是在另一个线程中执行该 Task，无论是通过直接 new 一个 Thread 还是通过线程池，执行的都是 run 方法。

### 核心方法：run

```java
public void run() {
    //新建任务，CAS替换runner为当前线程
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);//设置执行结果
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);//处理中断逻辑
    }
}
```

- 运行任务：如果任务状态为NEW状态，则利用CAS修改为当前线程。执行完毕调用set(result)方法设置执行结果。set(result)源码如下：

```java
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();//执行完毕，唤醒等待线程
    }
}
```

- 首先利用cas修改state状态为COMPLETING，设置返回结果，然后使用 lazySet(UNSAFE.putOrderedInt)的方式设置state状态为NORMAL。结果设置完毕后，调用finishCompletion()方法唤醒等待线程，源码如下：

```java
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {//移除等待线程
            for (;;) {//自旋遍历等待线程
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);//唤醒等待线程
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
    //任务完成后调用函数，自定义扩展
    done();

    callable = null;        // to reduce footprint
}
```

- 回到run方法，如果在 run 期间被中断，此时需要调用handlePossibleCancellationInterrupt方法来处理中断逻辑，确保任何中断(例如cancel(true))只停留在当前run或runAndReset的任务中，源码如下：

```java
private void handlePossibleCancellationInterrupt(int s) {
    //在中断者中断线程之前可能会延迟，所以我们只需要让出CPU时间片自旋等待
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            Thread.yield(); // wait out pending interrupt
}
```

### 核心方法：get

```java
//获取执行结果
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
```

FutureTask 通过get()方法获取任务执行结果。如果任务处于未完成的状态(`state <= COMPLETING`)，就调用awaitDone方法(后面单独讲解)等待任务完成。任务完成后，通过report方法获取执行结果或抛出执行期间的异常。report源码如下：

```java
//返回执行结果或抛出异常
private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

### 核心方法：awaitDone(boolean timed, long nanos)

```java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {//自旋
        if (Thread.interrupted()) {//获取并清除中断状态
            removeWaiter(q);//移除等待WaitNode
            throw new InterruptedException();
        }

        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;//置空等待节点的线程
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        else if (!queued)
            //CAS修改waiter
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);//超时，移除等待节点
                return state;
            }
            LockSupport.parkNanos(this, nanos);//阻塞当前线程
        }
        else
            LockSupport.park(this);//阻塞当前线程
    }
}
```

awaitDone 用于等待任务完成，或任务因为中断或超时而终止。返回任务的完成状态。函数执行逻辑如下：

```java
private void removeWaiter(WaitNode node) {
    if (node != null) {
        node.thread = null;//首先置空线程
        retry:
        for (;;) {          // restart on removeWaiter race
            //依次遍历查找
            for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                s = q.next;
                if (q.thread != null)
                    pred = q;
                else if (pred != null) {
                    pred.next = s;
                    if (pred.thread == null) // check for race
                        continue retry;
                }
                else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,q, s)) //cas替换
                    continue retry;
            }
            break;
        }
    }
}
```

- 加入当前线程状态为结束(state>COMPLETING)，则根据需要置空等待节点的线程，并返回 Future 状态；
- 如果当前状态为正在完成(COMPLETING)，说明此时 Future 还不能做出超时动作，为任务让出CPU执行时间片；
- 如果state为NEW，先新建一个WaitNode，然后CAS修改当前waiters；
- 如果等待超时，则调用removeWaiter移除等待节点，返回任务状态；如果设置了超时时间但是尚未超时，则park阻塞当前线程；
- 其他情况直接阻塞当前线程。

### 核心方法：cancel(boolean mayInterruptIfRunning)

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    //如果当前Future状态为NEW，根据参数修改Future状态为INTERRUPTING或CANCELLED
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {//可以在运行时中断
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();//移除并唤醒所有等待线程
    }
    return true;
}
```

尝试取消任务。如果任务已经完成或已经被取消，此操作会失败。

- 如果当前Future状态为NEW，根据参数修改Future状态为INTERRUPTING或CANCELLED。
- 如果当前状态不为NEW，则根据参数mayInterruptIfRunning决定是否在任务运行中也可以中断。中断操作完成后，调用finishCompletion移除并唤醒所有等待线程。

## 应用实例

### Future & ExecutorService

```java
public class FutureDemo {
      public static void main(String[] args) {
          ExecutorService executorService = Executors.newCachedThreadPool();
          Future future = executorService.submit(new Callable<Object>() {
              @Override
              public Object call() throws Exception {
                  Long start = System.currentTimeMillis();
                  while (true) {
                      Long current = System.currentTimeMillis();
                     if ((current - start) > 1000) {
                         return 1;
                     }
                 }
             }
         });
  
         try {
             Integer result = (Integer)future.get();
             System.out.println(result);
         }catch (Exception e){
             e.printStackTrace();
         }
     }
}
```

### FutureTask & ExecutorService

```java
ExecutorService executor = Executors.newCachedThreadPool();
Task task = new Task();
FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
executor.submit(futureTask);
executor.shutdown();
```

### Future & Thread

```java
import java.util.concurrent.*;
 
public class CallDemo {
 
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // 2. 新建FutureTask,需要一个实现了Callable接口的类的实例作为构造函数参数
        FutureTask<Integer> futureTask = new FutureTask<Integer>(new Task());
        // 3. 新建Thread对象并启动
        Thread thread = new Thread(futureTask);
        thread.setName("Task thread");
        thread.start();
 
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
 
        System.out.println("Thread [" + Thread.currentThread().getName() + "] is running");
 
        // 4. 调用isDone()判断任务是否结束
        if(!futureTask.isDone()) {
            System.out.println("Task is not done");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        int result = 0;
        try {
            // 5. 调用get()方法获取任务结果,如果任务没有执行完成则阻塞等待
            result = futureTask.get();
        } catch (Exception e) {
            e.printStackTrace();
        }
 
        System.out.println("result is " + result);
 
    }
 
    // 1. 继承Callable接口,实现call()方法,泛型参数为要返回的类型
    static class Task  implements Callable<Integer> {
 
        @Override
        public Integer call() throws Exception {
            System.out.println("Thread [" + Thread.currentThread().getName() + "] is running");
            int result = 0;
            for(int i = 0; i < 100;++i) {
                result += i;
            }
 
            Thread.sleep(3000);
            return result;
        }
    }
}
```

