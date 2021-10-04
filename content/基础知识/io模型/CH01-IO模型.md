---
type: docs
title: "Linux IO/线程 模型"
linkTitle: "Linux IO/线程 模型"
weight: 1
---

## 用户空间与内核空间

我们知道现在的操作系统都是采用虚拟存储器，那么对 32 位操作系统来说，它的寻址空间即虚拟存储空间为 4G，2 的 32 次方。操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。**为了保证用户进程不能直接操作内核，保证内核的的安全，操作系统将虚拟内存空间划分为两部分，一部分是内核空间，一部分是用户空间。** 

针对 Linux 操作系统而言，将最高的 1G 字节，即从虚拟地址 0xC0000000 到 0xFFFFFFFF 供内核使用，称为内核空间。而较低的 3G 字节，即从虚拟地址 0x00000000 到 0xBFFFFFFF，供进程使用，称为用户空间。每个进程都可以通过系统调用进入内核，因此 Linux 内核由系统内的所有进程共享。于是，**从具体进程的角度看，每个进程可以拥有 4G 字节的虚拟空间**。

有了用户空间和内核空间，整个 Linux 内部结构可以分为三个部分，从最底层到最上层依次是：**硬件、内核空间、用户空间**。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225144207.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

需要注意的细节是，从上图可以看出内核的组成：

- 内核空间中存放的是内核代码和数据，而进程的用户空间存放的是用户程序的代码和数据。不管是内核空间还是用户空间，都处于虚拟空间之中。
- Linux 使用两级保护机制：0 级供内核使用，3 级供用户程序使用。

## 服务端处理网络请求的流程

为了 OS 的安全性等考虑，进程是无法直接操作 IO 设备的，其**必须通过系统调用来请求内核以协助完成 IO 动作，而内核会为每个 IO 设备维护一个 buffer**。

整个请求过程为：

1. 用户进程发起请求；
2. 内核接收到请求后；
3. **从 IO 设备中获取数据到 buffer 中**；
4. **再将 buffer 中的数据 copy 到用户进程的地址空间**；
5. 该用户进程获取到数据后再响应客户端。

服务端处理网络请求的典型流程图如下：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225151943.png" style="display:block;width:80%;" alt="NAME" align=center /> </div>

在请求过程中，数据从 IO 设备输入至 buffer 需要时间，从 buffer 复制将数据复制到用户进程也需要时间。因此**根据在这两段时间内等待方式的不同，IO 动作可以分为以下五种**：

- 阻塞 IO，Blocking IO
- 非阻塞 IO，Non-Blocking IO
- IO 复用，IO Multiplexing
- 信号驱动的 IO，Signal Driven IO
- 异步 IO，Asynchrnous IO

> 更多细节参考 <Unix 网络编程>，6.2 节 “IO Models”。

设计服务端并发模型时，主要有如下两个关键点： 

- 服务器如何管理连接，获取请求数据。
- 服务器如何处理请求。

以上两个关键点最终都与操作系统的 I/O 模型以及线程(进程)模型相关，下面详细介绍这两个模型。

## 阻塞/非阻塞、同步/异步

### **阻塞/非阻塞**：

- 阻塞调用是指调用结果返回之前，当前线程会被挂起。调用线程只有在得到结果之后才会返回。
- 非阻塞调用指在不能立刻得到结果之前，该调用不会阻塞当前线程。

区别：

- 两者的最大区别在于被调用方在收到请求到返回结果之前的这段时间内，调用方是否一直在等待。
- 阻塞是指调用方一直在等待而且别的事情什么都不做；非阻塞是指调用方先去忙别的事情。

### **同步/异步**：

- 同步处理是指被调用方得到最终结果之后才返回给调用方；
- 异步处理是指被调用方先返回应答，然后再计算调用结果，计算完最终结果后再通知并返回给调用方。

### 区别与联系

**阻塞、非阻塞和同步、异步其实针对的对象是不一样的**：

- 阻塞、非阻塞的讨论对象是调用者。
- 同步、异步的讨论对象是被调用者。

## Linux 网络 I/O 模型

### recvfrom 函数

recvfrom 函数(经 Socket 接收数据)，这里把它视为系统调用。一个输入操作通常包括两个不同的阶段：

- 等待数据准就绪。
- 从内核向应用进程复制数据。

对于一个套接字上的输入操作，第一步通常涉及等待数据从网络中到达。当所等待分组到达时，它被复制到内核中的某个缓冲区。第二步就是把数据从内核缓冲区复制到应用进程缓冲区。

实际应用程序在通过系统调用完成上面的 2 步操作时，调用方式的阻塞、非阻塞，操作系统在处理应用程序请求时处理方式的同步、异步，可以分为 5 种 I/O 模型。

### 阻塞式 IO

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225202244.png" style="display:block;width:80%;" alt="NAME" align=center /> </div>

在阻塞式 IO 模型中，应用程序从调用 recvfrom 开始到它返回有数据报准备好这段时间是阻塞的，recvfrom 返回成功后，应用程序开始处理数据报。

- 优点：程序实现简单，在阻塞等待数据期间，进程、线程挂起，基本不会占用 CPU 资源。
- 每个连接需要独立的进程、线程单独处理，当并发请求量大时为了维护程序，内存、线程切换开销很大，这种模型在实际生产中很少使用。

### 非阻塞 IO

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225202544.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

在非阻塞 IO 模型中，应用程序把一个套接口设置为非阻塞，就是告诉内核，当所有请求的 IO 操作无法完成时，不要将进程睡眠。

而是返回一个错误，应用程序基于 IO 操作函数，将会不断的轮询数据是否已经准备就绪，直到数据准备就绪。

- 优点：不会阻塞在内核的等待数据过程，每次发起的 IO 请求可以立即返回，不会阻塞等待，实时性比较好。
- 缺点：轮询将会不断的询问内核，这将占用大量的 CPU 时间，系统资源利用率较低，所以一般 Web 服务器不会使用这种 IO 模型。

### IO 多路复用

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225202916.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

在 IO 复用模型中，会用到 Select、Poll、Epoll 函数，这些函数会使进程阻塞，但是和阻塞 IO 有所不同。

这些函数可以同时阻塞多个 IO 操作，而且可以同时对多个读、写操作的 IO 函数进行检测，直到有数据可读或可写时，才会真正调用 IO 操作函数。

- 优点：可以基于一个阻塞对象，同时在多个描述符上等待就绪，而不是使用多个线程(每个文件描述符一个线程)，这样可以大大节省系统资源。
- 当连接数较少时效率比“多线程+阻塞IO”的模式效率低，可能延迟更大，因为单个连接处理需要 2 次系统调用，占用时间会增加。

### 信号驱动 IO

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225203258.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

在信号驱动 IO 模型中，应用程序使用套接口进行信号驱动 IO，并安装一个信号处理函数，进程继续运行并不阻塞。

当数据准备好时，进程会收到一个 SIGIO 信号，可以在信号处理函数中调用 IO 操作函数处理数据。

- 优点：线程没有在等待数据时被阻塞，可以提高资源利用率。
- 缺点：信号 IO 模式在大量 IO 操作时可能会因为信号队列溢出而导致无法通知。

信号驱动 IO 尽管对于处理 UDP 套接字来说有用，即这种信号通知意味着到达了一个数据报，或者返回一个异步错误。

但是，对于 TCP 而言，信号驱动 IO 方式近乎无用。因为导致这种通知的条件为数众多，逐个进行判断会消耗很大的资源，与前几种方式相比优势尽失。

### 异步 IO

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225203813.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

由 POSIX 规范定义，应用程序告知内核启动某个操作，并让内核在整个操作完成后(包括将数据从内核拷贝到应用程序的缓冲区)通知应用程序。

这种模型与信号驱动模型的主要区别在于：信号驱动 IO 是由内核通知应用程序合适启动一个 IO 操作，而异步 IO 模型是由内核通知应用程序 IO 操作合适完成。

- 优点：异步 IO 能够充分利用 DMA 特性，让 IO 操作与计算重叠。
- 缺点：需要实现真正的异步 IO，操作系统需要做大量的工作。当前 Windows 下通过 IOCP 实现了真正的异步 IO。

而在 Linux 系统下直到 2.6 版本才引入，目前 AIO 并不完善，因此在 Linux 下实现并发网络编程时都是以 IO 复用模型为主。

### IO 模型对比

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225204240.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

从上图可以看出，越往后，阻塞越少，理论上效率也最优。

这五种模型中，前四种属于同步 IO，因为其中真正的 IO 操作(recvfrom 函数调用)将阻塞进程/线程，只有异步 IO 模型才与 POSIX 定义的异步 IO 相匹配。

## 进程/线程模型

介绍完服务器如何基于 IO 模型**管理连接、获取输入数据**，下面介绍服务器如何基于进程、线程模型来**处理请求**。

### 传统阻塞 IO 服务模型

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225204551.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

特点：

- 采用阻塞式 IO 模型获取输入数据。
- 每个连接都需要独立的线程完成数据输入的读取、业务处理、数据返回操作。

存在问题：

- 当请求的并发数较大时，需要创建大量线程来处理连接，系统资源占用较大。
- 当连接建立后，如果当前线程暂时没有数据可读，则线程就阻塞在 Read 操作上，造成线程资源浪费。

### Reactor 模式

针对传统阻塞 IO 服务模型的 2 个缺点，比较常见的有如下解决方案：

- **基于 IO 复用模型**，多个连接共用一个阻塞对象，应用程序只需要在一个阻塞对象上等待，无需阻塞等待所有连接。
  - 当某条连接有新的数据可处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理。
- **基于线程池复用线程资源**，不必再为每个连接创建线程，将连接完成后的业务处理任务分配给线程进行处理，一个线程可以多个连接的业务。

**IO 复用模式结合线程池**，就是 Reactor 模式的基本设计思想，如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225205123.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

Reactor 模式，是指通过一个或多个输入同时传递给服务器来处理服务请求的事件驱动处理模式。

服务端程序处理传入的多路请求，并将它们同步分派给请求对应的处理线程，Reactor 模式也叫 Dispatcher 模式。

即 IO 多路复用以统一的方式监听事件，收到事件后分发(Dispatch 给某线程)，是编写高性能服务器的必备技术之一。

Reactor 模式有两个关键组件构成：

- Reactor：在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序对 IO 事件做出反应。它就像公司的电话接线员，接听来自客户的电话并将线路转移给适当的联系人。
- Handlers：处理程序执行 IO 事件需要完成的实际组件，类似于客户想要与之交谈的客服坐席。Reactor 通过调度适当的处理程序来响应 IO 事件，处理程序执行非阻塞操作。

根据 Reactor 的数量和处理资源池线程的数量不同，有 3 种典型的实现：

- 单 Reactor 单线程
- 单 Reactor 多线程
- 主从 Reactor 多线程

#### 单 Reactor 单线程

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225205947.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

其中，Select 是前面 IO 复用模型介绍的标准网络编程 API，可以实现应用程序通过一个阻塞多向监听多路连接请求，其他方案的示意图也类似。

方案说明：

- Reactor 对象通过 Select 监听客户端请求事件，收到事件后通过 Dispatch 进行分发。
- 如果是“建立连接”请求事件，则由 Acceptor 通过 Accept 处理连接请求，同时创建一个 Handler 对象来处理连接完成后的后续业务处理。
- 如果不是“建立连接”事件，则 Reactor 会分发调用“连接”对应的 Handler 来响应。
- Handler 会完成 “Read->业务处理->Send” 的完整业务流程。

- 优点：模型简单，没有多线程、进程通信、竞争的问题，全部都在一个线程中完成。
- 缺点：性能问题，只有一个线程，无法完全发挥多个 CPU 的性能。Handler 在处理某个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈。

可靠性问题、线程意外跑飞、进入死循环，或导致整个系统的通信模块不可用，不能接收或处理外部消息，造成节点故障。

应用场景：客户端的数量有限，业务处理非常快，比如 Redis，业务处理的时间复杂度为 O(1)。

#### 单 Reactor 多线程

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225210742.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- Reactor 对象通过 Select 监控客户端请求事件，收到事件后通过 Dispatch 进行分发。
- 如果是建立连接请求事件，则由 Acceptor 通过 Accept 处理连接请求，同时创建一个 Handler 对象处理连接完成后续的各种事件。
- 如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来响应。
- Handler 只负责响应事件，不做具体业务处理，通过 Read 读取数据后，会分发给后面的 Worker 线程池进行业务处理。
- Worker 线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给 Handler 进行处理。
- Handler 收到响应结果后通过 Send 将响应结果返回给 Client。

- 优点：可以充分利用多核 CPU 的处理能力。
- 缺点：
  - 多线程数据共享和访问比较复杂；
  - Reactor 承担所有事件的监听和响应，在单线程中运行，高并发场景下容易成为性能瓶颈。

#### 主从 Reactor 多线程

针对单 Reactor 多线程模型中，Reactor 在单线程中运行，高并发场景下容易成为性能瓶颈，可以让 Reactor 在多线程中运行。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225213435.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- Reactor 主线程 MainReactor 对象通过 Select 监控建立连接事件，收到事件后通过 Acceptor 接收，处理建立连接事件。
- Acceptor 处理建立连接事件后，MainReactor 将连接分配 Reactor 子线程给 SubReactor 进行处理。
- SubReactor 将连接加入连接队列进行监听，并创建一个 Handler 用于处理各种连接事件。
- 当有新的事件发生时，SubReactor 会调用连接对应的 Handler 进行响应。
- Handler 通过 Read 读取数据后，会分发给后面的 Worker 线程池进行业务处理。
- Worker 线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给 Handler 进行处理。
- Handler 收到响应结果后通过 Send 将响应结果返回给 Client。

- 优点：父线程与子线程的数据交互简单、职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。

父线程与子线程的数据交互简单，Reactor 主线程只需要把新连接传递给子线程即可，子线程无需返回数据。

这种模型在很多项目中广泛使用，包括 Nginx 主从 Reactor 多线程模型，Memcached 主从多线程。

#### Reactor 模式总结

三种模式可以用一个比喻来理解：餐厅常常雇佣接待员负责迎接顾客，当顾客入座后，侍应生专门为这张桌子服务。

- 单 Reactor 单线程：接待员和侍应生是同一个人，全程为顾客服务。
- 单 Reactor 多线程：一个接待员、多个侍应生，接待员只负责接待。
- 主从 Reactor：多个接待员，多个侍应生。

Reactor 模式具有如下的优点：

- 响应快：不必为单个同步时间所阻塞，虽然 Reactor 本身依然是同步的。
- 编程相对简单：可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程的切换开销。
- 可扩展性：可以方便的通过增加 Reactor 实例个数来充分利用 CPU 资源。
- 可复用性：Reactor 模型本身与具体事件处理逻辑无关，具有很高的复用性。

### Proactor 模型

在 Reactor 模式中，Reactor 等待某个事件、可应用或操作的状态发生(比如文件描述符可读、Socket 可读写)。

然后把该事件传递给事先注册的 Handler(事件处理函数或回调函数)，由后者来做实际的读写操作。

其中的读写操作都需要应用程序同步操作，所以 **Reactor 是非阻塞同步网络模型**。

如果把 IO 操作改为异步，即交给操作系统来完成 IO 操作，就能进一步提升性能，这就是异步网络模型 Proactor。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190225214717.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

Proactor 是和异步 I/O 相关的，详细方案如下：

- ProactorInitiator 创建 Proactor 和 Handler 对象，并将 Proactor 和 Handler 都通过 AsyOptProcessor(Asynchronous Operation Processor) 注册到内核。
- AsyOptProcessor 处理注册请求，并处理 I/O 操作。
- AsyOptProcessor 完成 I/O 操作后通知 Proactor。
- Proactor 根据不同的事件类型回调不同的 Handler 进行业务处理。
- Handler 完成业务处理。

可以看出 Proactor 和 Reactor 的区别：

- Reactor 是在事件发生时就通知事先注册的事件(读写在应用程序线程中处理完成)。
- Proactor 是在事件发生时基于异步 I/O 完成读写操作(由内核完成)，待 I/O 操作完成后才回调应用程序的处理器来进行业务处理。

理论上 Proactor 比 Reactor 效率更高，异步 I/O 更加充分发挥 DMA(Direct Memory Access，直接内存存取)的优势，但是有如下缺点： 

- **编程复杂性**：由于异步操作流程的事件的初始化和事件完成在时间和空间上都是相互分离的，因此开发异步应用程序更加复杂。应用程序还可能因为反向的流控而变得更加难以 Debug。
- **内存使用**：缓冲区在读或写操作的时间段内必须保持住，可能造成持续的不确定性，并且每个并发操作都要求有独立的缓存，相比 Reactor 模式，在 Socket 已经准备好读或写前，是不要求开辟缓存的。
- **操作系统支持**，Windows 下通过 IOCP 实现了真正的异步 I/O，而在 Linux 系统下，Linux 2.6 才引入，目前异步 I/O 还不完善。

因此在 Linux 下实现高并发网络编程都是以 Reactor 模型为主。