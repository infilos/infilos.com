---
type: docs
title: "CH11-AIO原理"
linkTitle: "CH11-AIO原理"
weight: 11
---

## 异步IO

上面两篇文章中，我们分别讲解了阻塞式同步IO、非阻塞式同步IO、多路复用IO 这三种IO模型，以及JAVA对于这三种IO模型的支持。重点说明了IO模型是由操作系统提供支持，且这三种IO模型都是同步IO，都是采用的“应用程序不询问我，我绝不会主动通知”的方式。

异步IO则是采用“订阅-通知”模式: 即应用程序向操作系统注册IO监听，然后继续做自己的事情。当操作系统发生IO事件，并且准备好数据后，在主动通知应用程序，触发相应的函数:

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502141006.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

和同步IO一样，异步IO也是由操作系统进行支持的。微软的windows系统提供了一种异步IO技术: IOCP(I/O Completion Port，I/O完成端口)；

Linux下由于没有这种异步IO技术，所以使用的是epoll(上文介绍过的一种多路复用IO技术的实现)对异步IO进行模拟。

### JAVA AIO框架简析

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502141030.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

这里通过这个结构分析要告诉各位读者JAVA AIO中类设计和操作系统的相关性

在文中我们一再说明JAVA AIO框架在windows下使用windows IOCP技术，在Linux下使用epoll多路复用IO技术模拟异步IO，这个从JAVA AIO框架的部分类设计上就可以看出来。例如框架中，在Windows下负责实现套接字通道的具体类是“sun.nio.ch.WindowsAsynchronousSocketChannelImpl”，其引用的IOCP类型文档注释如是:

```java
/** 
* Windows implementation of AsynchronousChannelGroup encapsulating an I/O 
* completion port. 
*/
```

如果您感兴趣，当然可以去看看全部完整代码(建议从“java.nio.channels.spi.AsynchronousChannelProvider”这个类看起)。

特别说明一下，请注意图中的“java.nio.channels.NetworkChannel”接口，这个接口同样被JAVA NIO框架实现了，如下图所示:

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502141056.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### 要点讲解

注意在JAVA NIO框架中，我们说到了一个重要概念“selector”(选择器)。它负责代替应用查询中所有已注册的通道到操作系统中进行IO事件轮询、管理当前注册的通道集合，定位发生事件的通道等操操作；但是在JAVA AIO框架中，由于应用程序不是“轮询”方式，而是订阅-通知方式，所以不再需要“selector”(选择器)了，改由channel通道直接到操作系统注册监听。

JAVA AIO框架中，只实现了两种网络IO通道“AsynchronousServerSocketChannel”(服务器监听通道)、“AsynchronousSocketChannel”(socket套接字通道)。但是无论哪种通道他们都有独立的fileDescriptor(文件标识符)、attachment(附件，附件可以使任意对象，类似“通道上下文”)，并被独立的SocketChannelReadHandle类实例引用。我们通过debug操作来看看它们的引用结构:

在测试过程中，我们启动了两个客户端(客户端用什么语言来写都行，用阻塞或者非阻塞方式也都行，只要是支持 TCP Socket套接字的就行，然后我们观察服务器端对这两个客户端通道的处理情况:

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502141123.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

可以看到，在服务器端分别为客户端1和客户端2创建的两个WindowsAsynchronousSocketChannelImpl对象为:

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502141141.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

客户端1: WindowsAsynchronousSocketChannelImpl: 760 | FileDescriptor: 762

客户端2: WindowsAsynchronousSocketChannelImpl: 792 | FileDescriptor: 797

接下来，我们让两个客户端发送信息到服务器端，并观察服务器端的处理情况。客户端1发来的消息和客户端2发来的消息，在服务器端的处理情况如下图所示:

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502141156.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

客户端1: WindowsAsynchronousSocketChannelImpl: 760 | FileDescriptor: 762 | SocketChannelReadHandle: 803 | HeapByteBuffer: 808

客户端2: WindowsAsynchronousSocketChannelImpl: 792 | FileDescriptor: 797 | SocketChannelReadHandle: 828 | HeapByteBuffer: 833

可以明显看到，服务器端处理每一个客户端通道所使用的SocketChannelReadHandle(处理器)对象都是独立的，并且所引用的SocketChannel对象都是独立的。

JAVA NIO和JAVA AIO框架，除了因为操作系统的实现不一样而去掉了Selector外，其他的重要概念都是存在的，例如上文中提到的Channel的概念，还有演示代码中使用的Buffer缓存方式。实际上JAVA NIO和JAVA AIO框架您可以看成是一套完整的“高并发IO处理”的实现。

### 还有改进可能

当然，以上代码是示例代码，目标是为了让您了解JAVA AIO框架的基本使用。所以它还有很多改造的空间，例如:

在生产环境下，我们需要记录这个通道上“用户的登录信息”。那么这个需求可以使用JAVA AIO中的“附件”功能进行实现。

记住JAVA AIO 和 JAVA NIO 框架都是要使用线程池的(当然您也可以不用)，线程池的使用原则，一定是只有业务处理部分才使用，使用后马上结束线程的执行(还回线程池或者消灭它)。JAVA AIO框架中还有一个线程池，是拿给“通知处理器”使用的，这是因为JAVA AIO框架是基于“订阅-通知”模型的，“订阅”操作可以由主线程完成，但是您总不能要求在应用程序中并发的“通知”操作也在主线程上完成吧^_^。

最好的改进方式，当然就是使用Netty或者Mina咯

## 为什么还有Netty

- 那么有的读者可能就会问，既然JAVA NIO / JAVA AIO已经实现了各主流操作系统的底层支持，那么为什么现在主流的JAVA NIO技术会是Netty和MINA呢? 答案很简单: 因为更好用，这里举几个方面的例子:
- 虽然JAVA NIO 和 JAVA AIO框架提供了 多路复用IO/异步IO的支持，但是并没有提供上层“信息格式”的良好封装。例如前两者并没有提供针对 Protocol Buffer、JSON这些信息格式的封装，但是Netty框架提供了这些数据格式封装(基于责任链模式的编码和解码功能)
- 要编写一个可靠的、易维护的、高性能的(注意它们的排序)NIO/AIO 服务器应用。除了框架本身要兼容实现各类操作系统的实现外。更重要的是它应该还要处理很多上层特有服务，例如: 客户端的权限、还有上面提到的信息格式封装、简单的数据读取。这些Netty框架都提供了响应的支持。
- JAVA NIO框架存在一个poll/epoll bug: Selector doesn’t block on Selector.select(timeout)，不能block意味着CPU的使用率会变成100%(这是底层JNI的问题，上层要处理这个异常实际上也好办)。当然这个bug只有在Linux内核上才能重现。
- 这个问题在JDK 1.7版本中还没有被完全解决: http://bugs.java.com/bugdatabase/view_bug.do?bug_id=2147719。虽然Netty 4.0中也是基于JAVA NIO框架进行封装的(上文中已经给出了Netty中NioServerSocketChannel类的介绍)，但是Netty已经将这个bug进行了处理。

