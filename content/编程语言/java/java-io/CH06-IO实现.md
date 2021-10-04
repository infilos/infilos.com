---
type: docs
title: "CH06-IO实现"
linkTitle: "CH06-IO实现"
weight: 6
---

## 概览

> 说到 I/O，想必大家都不会陌生， I/O 英语全称：Input/Output，即**输入/输出**，通常**指数据在内部存储器和外部存储器或其他周边设备之间的输入和输出**。

比如我们常用的 **SD卡**、**U盘**、**移动硬盘**等等存储文件的硬件设备，当我们将其插入电脑的 usb 硬件接口时，我们就可以从电脑中读取设备中的信息或者写入信息，这个过程就涉及到 I/O 的操作。

当然，涉及 I/O 的操作，不仅仅局限于硬件设备的读写，还要网络数据的传输，比如，我们在电脑上用浏览器搜索互联网上的信息，这个过程也涉及到 I/O 的操作。

无论是从磁盘中读写文件，还是在网络中传输数据，可以说 I/O 主要为处理**人机交互**、**机与机交互**中获取和交换信息提供的一套解决方案。

在 Java 的 IO 体系中，类将近有 80 个，位于`java.io`包下，感觉很复杂，但是这些类大致可以分成四组：

- **基于字节操作的 I/O 接口：InputStream 和 OutputStream**
- **基于字符操作的 I/O 接口：Writer 和 Reader**
- **基于磁盘操作的 I/O 接口：File**
- **基于网络操作的 I/O 接口：Socket**

前两组主要从**传输数据的数据格式**不同，进行分组；后两组主要从**传输数据的方式**不同，进行分组。

虽然 Socket 类并不在` java.io`包下，但是我们仍然把它们划分在一起，因为 I/O 的核心问题，要么是数据格式影响 I/O 操作，要么是传输方式影响 I/O 操作，**也就是将什么样的数据写到什么地方的问题**，I/O 只是人与机器或者机器与机器交互的手段，除了在它们能够完成这个交互功能外，我们关注的就是如何提高它的运行效率了，而**数据格式**和**传输方式**是影响效率最关键的因素。

## 字节格式

基于字节的输入和输出操作接口分别是：InputStream 和 OutputStream 。

### 字节输入流

InputStream 输入流的类继承层次如下图所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502130438.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

输入流根据数据节点类型和处理方式，分别可以划分出了若干个子类，如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502130515.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

#### 字节输出流

OutputStream 输出流的类继承层次如下图所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502130542.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

输出流根据数据节点类型和处理方式，也分别可以划分出了若干个子类，如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502130602.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

在这里就不详细的介绍各个子类的使用方法，有兴趣的朋友可以查看 JDK 的 API 说明文档，笔者也会在后期的文章会进行详细的介绍，这里只是重点想说一下，**无论是输入还是输出，操作数据的方式可以组合使用，各个处理流的类并不是只操作固定的节点流**，比如如下输出方式：

```java
//将文件输出流包装到序列化输出流中，再将序列化输出流包装到缓冲中 OutputStream out = new BufferedOutputStream(new ObjectOutputStream(new FileOutputStream(new File("fileName")))； 
```

另外，**输出流最终写到什么地方必须要指定**，要么是写到硬盘中，要么是写到网络中，从图中可以发现，写网络实际上也是写文件，只不过写到网络中，需要经过底层操作系统将数据发送到其他的计算机中，而不是写入到本地硬盘中。

## 字符格式

**不管是磁盘还是网络传输，最小的存储单元都是字节，而不是字符**，所以 I/O 操作的都是字节而不是字符，但是为什么要有操作字符的 I/O 接口呢？

这是因为我们的程序中通常操作的数据都是以字符形式，**为了程序操作更方便而提供一个直接写字符的 I/O 接口，仅此而已。**

基于字符的输入和输出操作接口分别是：Reader 和 Writer ，下图是字符的 I/O 操作接口涉及到的类结构图。

#### 字符输入流

Reader 输入流的类继承层次如下图所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502130732.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

同样的，输入流根据数据节点类型和处理方式，分别可以划分出了若干个子类，如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502130744.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

#### 字符输出流

Writer 输出流的类继承层次如下图所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502130808.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

同样的，输出流根据数据节点类型和处理方式分类，分别可以划分出了若干个子类，如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502130821.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

不管是 Reader 还是 Writer 类，它们都只定义了读取或写入数据字符的方式，也就是说要么是读要么是写，但是并没有规定数据要写到哪去，写到哪去就是我们后面要讨论的基于磁盘或网络的工作机制。

### 字节与字符的转化

刚刚我们说到，不管是磁盘还是网络传输，最小的存储单元都是字节，而不是字符，设计字符的原因是为了程序操作更方便，那么怎么将字符转化成字节或者将字节转化成字符呢？

**InputStreamReader 和 OutputStreamWriter 就是转化桥梁。**

#### 输入流转化过程

输入流字符解码相关类结构的转化过程如下图所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/1786534ce9b3483cb9a5914ef4a227f9.jpg" style="display:block;width:50%;" alt="NAME" align=center /> </div>

从图上可以看到，InputStreamReader 类是字节到字符的转化桥梁， 其中`StreamDecoder`指的是一个**解码**操作类，`Charset`指的是字符集。

InputStream 到 Reader 的过程需要指定编码字符集，否则将采用操作系统默认字符集，很可能会出现乱码问题，StreamDecoder 则是完成字节到字符的解码的实现类。

打开源码部分，**InputStream 到 Reader 转化过程**，如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502131002.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

#### 输出流转化过程

输出流转化过程也是类似，如下图所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502131023.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

通过 OutputStreamWriter 类完成字符到字节的编码过程，由 `StreamEncoder` 完成**编码**过程。

源码部分，**Writer 到 OutputStream 转化过程**，如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502131039.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

## 磁盘传输

前面介绍了Java I/O 的操作接口，这些接口主要定义了如何操作数据，以及介绍了操作数据格式的方式：字节流和字符流。

**还有一个关键问题就是数据写到何处，其中一个主要的处理方式就是将数据持久化到物理磁盘。**

我们知道数据在磁盘的唯一最小描述就是文件，也就是说上层应用程序只能通过文件来操作磁盘上的数据，文件也是操作系统和磁盘驱动器交互的一个最小单元。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502131240.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- **在 Java I/O 体系中，File 类是唯一代表磁盘文件本身的对象**。
- File 类定义了一些与平台无关的方法来操作文件，包括检查一个**文件是否存在、创建、删除文件、重命名文件、判断文件的读写权限是否存在、设置和查询文件的最近修改时间**等等操作。

值得注意的是 Java 中通常的 File 并不代表一个真实存在的文件对象，当你通过指定一个路径描述符时，它就会返回一个代表这个路径相关联的一个**虚拟对象**，这个可能是一个真实存在的文件或者是一个包含多个文件的目录。

例如，读取一个文件内容，程序如下：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502131318.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

以上面的程序为例，从硬盘中读取一段文本字符，操作流程如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502131340.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

当我们传入一个指定的文件名来创建 File 对象，通过 FileReader 来读取文件内容时，会自动创建一个`FileInputStream`对象来读取文件内容，也就是我们上文中所说的字节流来读取文件。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502131455.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

紧接着，会创建一个`FileDescriptor`的对象，其实这个对象就是真正代表一个存在的文件对象的描述。可以通过`FileInputStream`对象调用`getFD() `方法获取真正与底层操作系统关联的文件描述。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502131526.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

由于我们需要读取的是字符格式，所以需要 `StreamDecoder` 类将`byte`解码为`char`格式，至于如何从磁盘驱动器上读取一段数据，由操作系统帮我们完成。

## 网络传输

继续来说说数据写到何处的另一种处理方式：**将数据写入互联网中以供其他电脑能访问**。

### Socket简介

在现实中，Socket 这个概念没有一个具体的实体，它是描述计算机之间完成相互通信一种抽象定义。

打个比方，可以把 Socket 比作为两个城市之间的交通工具，有了它，就可以在城市之间来回穿梭了。并且，交通工具有多种，每种交通工具也有相应的交通规则。Socket 也一样，也有多种。大部分情况下我们使用的都是基于 TCP/IP 的流套接字，它是一种稳定的通信协议。

典型的基于 Socket 通信的应用程序场景，如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502132350.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

主机 A 的应用程序要想和主机 B 的应用程序通信，必须通过 Socket 建立连接，而建立 Socket 连接必须需要底层 TCP/IP 协议来建立 TCP 连接。

### 建立通信链路

我们知道网络层使用的 IP 协议可以帮助我们根据 IP 地址来找到目标主机，但是一台主机上可能运行着多个应用程序，如何才能与指定的应用程序通信就要通过 TCP 或 UPD 的地址也就是端口号来指定。这样就可以通过一个 Socket 实例代表唯一一个主机上的一个应用程序的通信链路了。

为了准确无误地把数据送达目标处，**TCP 协议采用了三次握手策略**，如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502132449.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- SYN 全称为 Synchronize Sequence Numbers，**表示同步序列编号**，是 TCP/IP 建立连接时使用的握手信号。

- ACK 全称为 Acknowledge character，即确认字符，**表示发来的数据已确认接收无误**。

在客户机和服务器之间建立正常的 TCP 网络连接时，客户机首先发出一个 **SYN 消息**，服务器使用 **SYN + ACK** 应答表示接收到了这个消息，最后客户机再以 **ACK** 消息响应。

这样在客户机和服务器之间才能建立起可靠的 TCP 连接，数据才可以在客户机和服务器之间传递。

- 发送端 –（发送带有 SYN 标志的数据包 ）–> 接受端（第一次握手）；
- 接受端 –（发送带有 SYN + ACK 标志的数据包）–> 发送端（第二次握手）；
- 发送端 –（发送带有 ACK 标志的数据包） –> 接受端（第三次握手）；

完成三次握手之后，客户端应用程序与服务器应用程序就可以开始传送数据了。

### 传输数据

当客户端要与服务端通信时，客户端首先要创建一个 Socket 实例，默认操作系统将为这个 Socket 实例分配一个没有被使用的本地端口号，并创建一个包含本地、远程地址和端口号的套接字数据结构，这个数据结构将一直保存在系统中直到这个连接关闭。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502132616.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

与之对应的服务端，也将创建一个 ServerSocket 实例，ServerSocket 创建比较简单，只要指定的端口号没有被占用，一般实例创建都会成功，同时操作系统也会为 ServerSocket 实例创建一个底层数据结构，这个数据结构中包含指定监听的端口号和包含监听地址的通配符，通常情况下都是`*`即监听所有地址。

之后当调用 accept() 方法时，将进入阻塞(等待)状态，等待客户端的请求。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502132704.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

我们先启动服务端程序，再运行客户端，服务端收到客户端发送的信息，服务端打印结果如下：

```
服务端收到客户端发送的消息：Hello，我是客户端！
```

注意，客户端只有与服务端建立三次握手成功之后，才会发送数据，而 TCP/IP 握手过程，底层操作系统已经帮我们实现了！

1. 当连接已经建立成功，服务端和客户端都会拥有一个 Socket 实例，每个 Socket 实例都有一个 **InputStream** 和 **OutputStream**，正如我们前面所说的，网络 I/O 都是以字节流传输的，Socket 正是通过这两个对象来交换数据。
2. 当 Socket 对象创建时，操作系统将会为 InputStream 和 OutputStream 分别分配一定大小的缓冲区，数据的写入和读取都是通过这个缓存区完成的。
3. 写入端将数据写到 OutputStream 对应的 SendQ 队列中，当队列填满时，数据将被发送到另一端 InputStream 的 RecvQ 队列中，如果这时 RecvQ 已经满了，那么 OutputStream 的 write 方法将会阻塞直到 RecvQ 队列有足够的空间容纳 SendQ 发送的数据。

值得特别注意的是，缓存区的大小以及写入端的速度和读取端的速度非常影响这个连接的数据传输效率，由于可能会发生阻塞，所以网络 I/O 与磁盘 I/O 在数据的写入和读取还要有一个协调的过程，如果两边同时传送数据时可能会产生死锁的问题。

**如何提高网络 IO 传输效率、保证数据传输的可靠，已经成了工程师们急需解决的问题。**

### IO 工作方式

在计算机中，IO 传输数据有三种工作方式，分别是 **BIO、NIO、AIO**。

在讲解 **BIO、NIO、AIO** 之前，我们先来回顾一下这几个概念：**同步与异步，阻塞与非阻塞**。

**同步与异步的区别**

- 同步就是发起一个请求后，接受者未处理完请求之前，不返回结果。
- 异步就是发起一个请求后，立刻得到接受者的回应表示已接收到请求，但是接受者并没有处理完，接受者通常依靠事件回调等机制来通知请求者其处理结果。

**阻塞和非阻塞的区别**

- 阻塞就是请求者发起一个请求，一直等待其请求结果返回，也就是当前线程会被挂起，无法从事其他任务，只有当条件就绪才能继续。
- 非阻塞就是请求者发起一个请求，不用一直等着结果返回，可以先去干其他事情，当条件就绪的时候，就自动回来。

而我们要讲的 **BIO、NIO、AIO** 就是**同步与异步、阻塞与非阻塞**的组合。

- BIO：同步阻塞 IO；
- NIO：同步非阻塞 IO；
- AIO：异步非阻塞 IO;

### BIO

BIO 俗称同步阻塞 IO，一种非常传统的 IO 模型，比如我们上面所举的那个程序例子，就是一个典型的**同步阻塞 IO **的工作方式。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502133132.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

采用 BIO 通信模型的服务端，通常由一个独立的 **Acceptor** 线程负责监听客户端的连接。

我们一般在服务端通过`while(true)`循环中会调用`accept() `方法等待监听客户端的连接，一旦接收到一个连接请求，就可以建立通信套接字进行读写操作，此时不能再接收其他客户端连接请求，只能等待同当前连接的客户端的操作执行完成， 不过可以通过多线程来支持多个客户端的连接。

**客户端多线程操作，程序如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502133226.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**服务端多线程操作，程序如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502133258.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

服务端运行结果，如下：

```
服务端收到客户端发送的消息：Hello，我是第 2 个，客户端！
服务端收到客户端发送的消息：Hello，我是第 4 个，客户端！
服务端收到客户端发送的消息：Hello，我是第 3 个，客户端！
服务端收到客户端发送的消息：Hello，我是第 0 个，客户端！
服务端收到客户端发送的消息：Hello，我是第 1 个，客户端！
```

如果要让 BIO 通信模型能够同时处理多个客户端请求，就必须使用多线程，也就是说它在接收到客户端连接请求之后为每个客户端创建一个新的线程进行链路处理，处理完成之后，通过输出流返回应答给客户端，线程销毁。

这就是典型的**一请求一应答**通信模型 。

如果出现100、1000、甚至10000个用户同时访问服务器，这个时候，如果使用这种模型，那么服务端也会创建与之相同的线程数量，**线程数急剧膨胀可能会导致线程堆栈溢出、创建新线程失败等问题，最终导致进程宕机或者僵死，不能对外提供服务**。

当然，我们可以通过使用 Java 中 ThreadPoolExecutor 线程池机制来改善，让线程的创建和回收成本相对较低，保证了系统有限的资源的控制，**实现了 N （客户端请求数量）大于 M （处理客户端请求的线程数量）的伪异步 I/O 模型。**

### 伪异步 BIO

为了解决同步阻塞 I/O 面临的一个链路需要一个线程处理的问题，后来有人对它的线程模型进行了优化，后端通过一个线程池来处理多个客户端的请求接入，形成客户端个数 M：线程池最大线程数 N 的比例关系，其中 M 可以远远大于 N，通过线程池可以灵活地调配线程资源，设置线程的最大值，防止由于海量并发接入导致资源耗尽。

伪异步IO模型图，如下图：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502133620.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

采用线程池和任务队列可以实现一种叫做伪异步的 I/O 通信框架，当有新的客户端接入时，将客户端的 Socket 封装成一个 Task 投递到后端的线程池中进行处理。

Java 的线程池维护一个消息队列和 N 个活跃线程，对消息队列中的任务进行处理。

**客户端，程序如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502133717.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**服务端，程序如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502133731.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**先启动服务端程序，再启动客户端程序，看看运行结果！**

**服务端，运行结果如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502133834.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**客户端，运行结果如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502133853.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

本例中测试的客户端数量是 30，服务端使用 java 线程池来处理任务，线程数量为 5 个，服务端不用为每个客户端都创建一个线程，由于线程池可以设置消息队列的大小和最大线程数，因此，它的资源占用是可控的，无论多少个客户端并发访问，都不会导致资源的耗尽和宕机。

在活动连接数不是特别高的情况下，这种模型是还不错，可以让每一个连接专注于自己的 I/O 并且编程模型简单，也不用过多考虑系统的过载、限流等问题。

但是，它的底层仍然是同步阻塞的 BIO 模型，当面对十万甚至百万级连接的时候，传统的 BIO 模型真的是无能为力的，我们需要一种更高效的 I/O 处理模型来应对更高的并发量。

### NIO

- NIO 中的 N 可以理解为 **Non-blocking**，一种同步非阻塞的 I/O 模型，在 Java 1.4 中引入，对应的在`java.nio`包下。

- NIO 新增了 **Channel、Selector、Buffer** 等抽象概念，支持面向缓冲、基于通道的 I/O 操作方法。

- NIO 提供了与传统 BIO 模型中的 `Socket` 和 `ServerSocket` 相对应的 `SocketChannel` 和 `ServerSocketChannel` 两种不同的套接字通道实现。

- NIO 这两种通道都支持**阻塞和非阻塞**两种模式。阻塞模式使用就像传统中的支持一样，比较简单，但是性能和可靠性都不好；**非阻塞模式正好与之相反**。

- 对于低负载、低并发的应用程序，可以使用同步阻塞 I/O 来提升开发效率和更好的维护性；
- 对于高负载、高并发的（**网络**）应用，应使用 NIO 的非阻塞模式来开发。

我们先看一下 NIO 涉及到的核心关联类图，如下：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502134125.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

上图中有三个关键类：**Channel 、Selector 和 Buffer**，它们是 NIO 中的核心概念。

- **Channel：可以理解为通道；**
- **Selector：可以理解为选择器；**
- **Buffer：可以理解为数据缓冲流；**

我们还是用前面的城市交通工具来继续形容 NIO 的工作方式，这里的 **Channel** 要比 **Socket** 更加具体，它可以比作为某种具体的交通工具，如汽车或是高铁、飞机等，而 **Selector** 可以比作为一个车站的车辆运行调度系统，它将负责监控每辆车的当前运行状态：是已经出站还是在路上等等，也就是说它可以轮询每个 **Channel** 的状态。

还有一个 **Buffer** 类，你可以将它看作为 IO 中 **Stream**，但是它比 IO 中的 **Stream** 更加具体化，我们可以将它比作为车上的座位，**Channel** 如果是汽车的话，那么 **Buffer** 就是汽车上的座位，**Channel** 如果是高铁，那么 **Buffer** 就是高铁上的座位，它始终是一个具体的概念，这一点与 **Stream** 不同。

**Socket 中的 Stream** 只能代表是一个座位，至于是什么座位由你自己去想象，也就是说你在上车之前并不知道这个车上是否还有没有座位，也不知道上的是什么车，因为你并不能选择，这些信息都已经被封装在了运输工具（**Socket**）里面了。

NIO 引入了 **Channel、Buffer 和 Selector** 就是想把 IO 传输过程中涉及到的**信息具体化**，让程序员有机会去控制它们。

当我们进行传统的网络 IO 操作时，比如调用 write() 往 Socket 中的 SendQ 队列写数据时，当一次写的数据超过 SendQ 长度时，操作系统会按照 SendQ 的长度进行分割的，这个过程中需要将用户空间数据和内核地址空间进行切换，而这个切换不是程序员可以控制的，由底层操作系统来帮我们处理。

而在 Buffer 中，我们可以控制 Buffer 的 capacity（容量），并且是否扩容以及如何扩容都可以控制。

还是以上面的操作为例子，为了方便观看结果，本次的客户端线程请求数改成15个。

**客户端，程序如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502134512.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**服务端，程序如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502134546.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**先启动服务端程序，再启动客户端程序，看看运行结果！**

**服务端，运行结果如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502134718.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**客户端，运行结果如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502134732.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**当然，客户端也不仅仅只限制于 IO 的写法，还可以使用`SocketChannel `来操作客户端，程序如下：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502134752.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**从操作上可以看到，NIO 的操作比传统的 IO 操作要复杂的多！**

**Selector** 被称为**选择器** ，当然你也可以翻译为**多路复用器** 。它是Java NIO 核心组件中的一个，用于检查一个或多个 **Channel**（通道）的状态是否处于**连接就绪**、**接受就绪**、**可读就绪**、**可写就绪**。

如此可以实现单线程管理多个 **channels**，也就是可以管理多个网络连接。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502134831.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**使用 Selector 的好处在于：** 相比传统方式使用多个线程来管理 IO，Selector 使用了更少的线程就可以处理通道了，并且实现网络高效传输！

虽然 java 中的 nio 传输比较快，为什么大家都不愿意用 JDK 原生 NIO 进行开发呢？

从上面的代码中大家都可以看出来，除了编程复杂、编程模型难之外，还有几个让人诟病的问题：

- **JDK 的 NIO 底层由 epoll 实现，该实现饱受诟病的空轮询 bug 会导致 cpu 飙升 100%！**
- **项目庞大之后，自行实现的 NIO 很容易出现各类 bug，维护成本较高！**

**但是，Google 的 Netty 框架的出现，很大程度上改善了 JDK 原生 NIO 所存在的一些让人难以忍受的问题**。

### AIO

最后就是 AIO 了，全称 Asynchronous I/O，可以理解为异步 IO，也被称为 NIO 2，在 Java 7 中引入了 NIO 的改进版 NIO 2，它是异步非阻塞的 IO 模型，也就是我们现在所说的 AIO。

异步 IO 是**基于事件和回调机制**实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。

**客户端，程序示例：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502135052.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**服务端，程序示例：**

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502135109.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

这种组合方式用起来比较复杂，只有在一些非常复杂的分布式情况下使用，像集群之间的消息同步机制一般用这种 I/O 组合方式。如 Cassandra 的 Gossip 通信机制就是采用异步非阻塞的方式。

## 参考资料

- http://www.justdojava.com/2019/12/18/java-io-1/
- http://ifeve.com/java-io/
- http://ifeve.com/java-nio-all/

