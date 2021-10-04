---
type: docs
title: "CH32-AllPools"
linkTitle: "CH32-AllPools"
weight: 32
---

## 七大属性

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430143504.png" style="display:block;width:90%;" alt="NAME" align=center /> </div>

- corePoolSize(int)：核心线程数量。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到任务队列当中。线程池将长期保证这些线程处于存活状态，即使线程已经处于闲置状态。除非配置了allowCoreThreadTimeOut=true，核心线程数的线程也将不再保证长期存活于线程池内，在空闲时间超过keepAliveTime后被销毁。
- workQueue：阻塞队列，存放等待执行的任务，线程从workQueue中取任务，若无任务将阻塞等待。当线程池中线程数量达到corePoolSize后，就会把新任务放到该队列当中。JDK提供了四个可直接使用的队列实现，分别是：基于数组的有界队列ArrayBlockingQueue、基于链表的无界队列LinkedBlockingQueue、只有一个元素的同步队列SynchronousQueue、优先级队列PriorityBlockingQueue。在实际使用时一定要设置队列长度。
- maximumPoolSize(int)：线程池内的最大线程数量，线程池内维护的线程不得超过该数量，大于核心线程数量小于最大线程数量的线程将在空闲时间超过keepAliveTime后被销毁。当阻塞队列存满后，将会创建新线程执行任务，线程的数量不会大于maximumPoolSize。
- keepAliveTime(long)：线程存活时间，若线程数超过了corePoolSize，线程闲置时间超过了存活时间，该线程将被销毁。除非配置了allowCoreThreadTimeOut=true，核心线程数的线程也将不再保证长期存活于线程池内，在空闲时间超过keepAliveTime后被销毁。
- TimeUnit unit：线程存活时间的单位，例如TimeUnit.SECONDS表示秒。
- RejectedExecutionHandler：拒绝策略，当任务队列存满并且线程池个数达到maximunPoolSize后采取的策略。ThreadPoolExecutor中提供了四种拒绝策略，分别是：抛RejectedExecutionException异常的AbortPolicy(如果不指定的默认策略)、使用调用者所在线程来运行任务CallerRunsPolicy、丢弃一个等待执行的任务，然后尝试执行当前任务DiscardOldestPolicy、不动声色的丢弃并且不抛异常DiscardPolicy。项目中如果为了更多的用户体验，可以自定义拒绝策略。
- threadFactory：创建线程的工厂，虽说JDK提供了线程工厂的默认实现DefaultThreadFactory，但还是建议自定义实现最好，这样可以自定义线程创建的过程，例如线程分组、自定义线程名称等。

## 工作原理

1. 通过execute方法提交任务时，当线程池中的线程数小于corePoolSize时，新提交的任务将通过创建一个新线程来执行，即使此时线程池中存在空闲线程。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430143621.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

2. 通过execute方法提交任务时，当线程池中线程数量达到corePoolSize时，新提交的任务将被放入workQueue中，等待线程池中线程调度执行。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/640-20210430143652230.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

3. 通过execute方法提交任务时，当workQueue已存满，且maximumPoolSize大于corePoolSize时，新提交的任务将通过创建新线程执行。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430143749.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

4. 当线程池中的线程执行完任务空闲时，会尝试从workQueue中取头结点任务执行。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430144336.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

5. 通过execute方法提交任务，当线程池中线程数达到maxmumPoolSize，并且workQueue也存满时，新提交的任务由RejectedExecutionHandler执行拒绝操作。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430144351.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

6. 当线程池中线程数超过corePoolSize，并且未配置allowCoreThreadTimeOut=true，空闲时间超过keepAliveTime的线程会被销毁，保持线程池中线程数为corePoolSize。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430144406.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

7. 当设置allowCoreThreadTimeOut=true时，任何空闲时间超过keepAliveTime的线程都会被销毁。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430144137.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

## 状态切换

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430144520.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210430144530.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

