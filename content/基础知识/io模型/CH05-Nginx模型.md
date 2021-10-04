---
type: docs
title: "Nginx IO 模型"
linkTitle: "Nginx IO 模型"
weight: 5
---

Nginx以其高性能，稳定性，丰富的功能，简单的配置和低资源消耗而闻名。本文从底层原理分析Nginx为什么这么快!

## 进程模型

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210505155303.png" style="display:block;width:70%;" alt="NAME" align=center /> </div>

Nginx 服务器在运行过程中：

-   **多进程**：一个 Master 进程、多个 Worker 进程。
-   Master进程：管理 Worker 进程。
    -   对外接口：接收外部的操作(信号);
    -   对内转发：根据外部的操作的不同，通过信号管理 Worker;
    -   监控：监控 Worker 进程的运行状态，Worker 进程异常终止后，自动重启 Worker 进程。
-   **Worker进程**：所有 Worker 进程都是平等的。
    -   处理网络请求，由Worker进程处理。
    -   Worker进程数量：在nginx.conf中配置，一般设置为核心数，充分利用 CPU 资源。
    -   同时，避免进程数量过多，避免进程竞争 CPU 资源，增加上下文切换的损耗。

### 思考

-   请求是连接到 Nginx，Master 进程负责处理和转发?
-   如何选定哪个 Worker 进程处理请求?请求的处理结果，是否还要经过 Master 进程?

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210505155459.png" style="display:block;width:70%;" alt="NAME" align=center /> </div>

## 请求处理过程

HTTP 连接建立和请求处理过程如下：

-   Nginx 启动时，Master 进程，加载配置文件。
-   Master 进程，初始化监听的 Socket。
-   Master 进程，Fork 出多个 Worker 进程。
-   Worker 进程，竞争新的连接，获胜方通过三次握手，建立 Socket 连接，并处理请求。

## 高性能高并发

Nginx 为什么拥有高性能并且能够支撑高并发?

-   Nginx采用多进程+异步非阻塞方式(IO 多路复用 Epoll)。
-   请求的完整过程：建立连接→读取请求→解析请求→处理请求→响应请求。
-   请求的完整过程对应到底层就是：读写Socket事件。

## 事件处理模型

Request：Nginx中HTTP请求。

基本的HTTP Web Server工作模式：

-   接收请求：逐行读取请求行和请求头，判断段有请求体后，读取请求体。
-   处理请求：获取对应的资源
-   返回响应：根据处理结果，生成相应的 HTTP 请求(响应行、响应头、响应体)。

Nginx也是这个套路，整体流程一致：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210505155640.png" style="display:block;width:70%;" alt="NAME" align=center /> </div>

## 模块化体系结构

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210505155803.png" style="display:block;width:70%;" alt="NAME" align=center /> </div>

Nginx的模块根据其功能基本上可以分为以下几种类型：

-   **event module**：
    -   搭建了独立于操作系统的事件处理机制的框架，及提供了各具体事件的处理。包括 ngx_events_module，ngx_event_core_module 和 ngx_epoll_module 等。
    -   Nginx 具体使用何种事件处理模块，这依赖于具体的操作系统和编译选项。
-   **phase handler**：
    -   此类型的模块也被直接称为 handler 模块。主要负责处理客户端请求并产生待响应内容，比如 ngx_http_static_module 模块，负责客户端的静态页面请求处理并将对应的磁盘文件准备为响应内容输出。
-   **output filter**：
    -   也称为 filter 模块，主要是负责对输出的内容进行处理，可以对输出进行修改。
    -   例如: 可以实现对输出的所有 html 页面增加预定义的 footbar 一类的工作，或者对输出的图片的 URL 进行替换之类的工作。
-   **upstream**：
    -   upstream 模块实现反向代理的功能，将真正的请求转发到后端服务器上，并从后端服务器上读取响应，发回客户端。
    -   upstream 模块是一种特殊的 handler，只不过响应内容不是真正由自己产生的，而是从后端服务器上读取的。
-   **load-balancer**：
    -   负载均衡模块，实现特定的算法，在众多的后端服务器中，选择一个服务器出来作为某个请求的转发服务器。

## Nginx vs Apache

Nginx：

-   IO 多路复用，Epoll(freebsd 上是 kqueue)
-   高性能
-   高并发
-   占用系统资源少

Apache：

-   阻塞+多进程/多线程
-   更稳定，Bug 少
-   模块更丰富

## 最大连接数

基础背景：

-   Nginx 是多进程模型，Worker 进程用于处理请求。
-   单个进程的连接数(文件描述符 fd)，有上限(nofile)：ulimit -n。
-   Nginx 上配置单个 Worker 进程的最大连接数：worker_connections 上限为 nofile。
-   Nginx 上配置 Worker 进程的数量：worker_processes。

因此，Nginx 的最大连接数：

-   Nginx 作为通用服务器时，最大的连接数：Worker进程数量 * 单个Worker进程的最大连接数。

-   Nginx 作为反向代理服务器时，能够服务的最大连接数：(Worker 进程数量 * 单个 Worker 进程的最大连接数)/ 2。

    >   Nginx 反向代理时，会建立 Client 的连接和后端 Web Server 的连接，占用 2 个连接。

