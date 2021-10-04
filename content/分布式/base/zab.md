---
type: docs
title: "ZAB"
linkTitle: "ZAB"
weight: 11
---

## 背景

Zookeeper 使用了一种称为 Zab（Zookeeper Atomic Broadcast）的协议作为其一致性复制的核心，据其作者说这是一种新发算法，其特点是充分考虑了 Yahoo 的具体情况：高吞吐量、低延迟、健壮、简单，但不过分要求其扩展性。

Zookeeper 的实现是有 Client、Server 构成，Server 端提供了一个一致性复制、存储服务，Client 端会提供一些具体的语义，比如分布式锁、选举算法、分布式互斥等。从存储内容来说，Server 端更多的是存储一些数据的状态，而非数据内容本身，因此 Zookeeper 可以作为一个小文件系统使用。数据状态的存储量相对不大，完全可以全部加载到内存中，从而极大地消除了通信延迟。

Server 可以 Crash 后重启，考虑到容错性，Server 必须“记住”之前的数据状态，因此数据需要持久化，但吞吐量很高时，磁盘的 IO 便成为系统瓶颈，其解决办法是使用缓存，把随机写变为连续写。

考虑到 Zookeeper 主要操作数据的状态，为了保证状态的一致性， Zookeeper 提出了两个安全属性（Safety Property）：

- 全序(total-order)，如果消息 A 在消息 B 之前发送，则所有 Server 应该看到相同顺序的结果。
- 因果关系(causal-order)，如果消息 A 在消息 B 之前发生(A 导致了 B)，并被一起发送，则 A 始终在 B 之前执行。

为了保证上述两个安全属性，Zookeeper 使用了 TCP 协议和 Leader 机制。通过使用 TCP 协议保证了消息的全序特性（先发先到），通过 Leader 机制解决了因果顺序问题：先到 Leader 的先执行。因为有了 Leader，Zookeeper 的架构就变为：Master-Slave 模式，但在该模式中Master（Leader）会 Crash，因此，Zookeeper 引入了 Leader 选举算法，以保证系统的健壮性。归纳起来 Zookeeper 整个工作分两个阶段：

1. Atomic Broadcast
2. Leader 选举

## Atomic Broadcast

同一时刻存在一个 Leader 节点，其他节点称为 Follower。如果是更新请求，如果客户端连接到 Leader 节点，则由 Leader 节点执行其请求；如果连接到 Follower 节点，则需转发到 Leader 节点执行。但对于读请求，Client 可以直接从 Follower 节点读取数据，如果需要读取到最新数据，则需要从 Leader 节点读取，Zookeeper 设计的读写比例是 2：1。

Leader 通过一个简化版的 2PC 模式向其他 Follower 发送请求，但与 2PC 有两个不同之处：

- 因为只有一个 Leader，Leader 提交到 Follower 的请求一定会被接受(没有其他 Leader 干扰)。
- 不需要所有 Follower 都响应成功，只要多数响应即可。

通俗的说，如果有 2f+1 个节点，允许 f 个节点失败。因为任何两个过半数集必要一个交集，当 Leader 切换时，通过这些交集节点可以获得当前系统的最新状态。如果没有一个过半数集存在(存活节点少于 f+1)则算法过程结束。

但又一个特例：如果 ABC 三个节点，A 是 Leader，如果 B 宕机，则 AC 能够正常工作，因为 A 是 Leader，AC 还能构成过半数集；如果 A 宕机则无法继续工作，因为用于 Leader 选举的过半数集无法构成。

## Leader Election

Leader 选举主要是依赖 Paxos 算法，具体算法过程请参考其他博文，这里仅考虑 Leader 选举带来的一些问题。

Leader 选举遇到的最大问题是，”新老交互“的问题，新 Leader 是否要继续老 Leader 的状态。这里要按老 Leader Crash 的时机点分几种情况：

1. 老 Leader 在 COMMIT 前 Crash（已经提交到本地）。
2. 老 Leader 在 COMMIT 后 Crash，但有部分 Follower 接收到了Commit请求。

第一种情况，这些数据只有老 Leader 自己知道，当老 Leader 重启后，需要与新 Leader 同步并把这些数据从本地删除，以维持状态一致。

第二种情况，新 Leader 应该能通过一个多数派获得老 Leader 提交的最新数据。老 Leader 重启后，可能还会认为自己是 Leader，可能会继续发送未完成的请求，从而因为两个 Leader 同时存在导致算法过程失败，解决办法是把 Leader 信息加入每条消息的 id 中，Zookeeper 中称为 zxid，zxid 为一 64 位数字，高 32 位为 leader 信息又称为 epoch，每次 leader 转换时递增；低 32 位为消息编号，Leader 转换时应该从 0 重新开始编号。通过 zxid，Follower 能很容易发现请求是否来自老 Leader，从而拒绝老 Leader 的请求。

因为在老 Leader 中存在着数据删除（情况1），因此 Zookeeper 的数据存储要支持补偿操作，这也就需要像数据库一样记录 log。

## ZAB 与 Paxos

Zab 的作者认为 Zab 与 Paxos 并不相同，只所以没有采用 Paxos 是因为 Paxos 保证不了全序顺序：

> Because multiple leaders can propose a value for a given instance two problems arise.
> First, proposals can conflict. Paxos uses ballots to detect and resolve conflicting proposals. 
> Second, it is not enough to know that a given instance number has been committed, processes must also be able to figure out which value has been committed.
> 

Paxos 算法的确是不关心请求之间的逻辑顺序，而只考虑数据之间的全序，但很少有人直接使用 Paxos 算法，都会经过一定的简化、优化。

一般 Paxos 都会有几种简化形式，其中之一便是，在存在 Leader 的情况下，可以简化为 1 个阶段（Leader Election）。仅有一个阶段的场景需要有一个健壮的 Leader，因此工作重点就变为 Leader 选举，在考虑到 Learner 的过程，还需要一个”学习“的阶段，通过这种方式，Paxos 可简化为两个阶段：

1. Leader Election
2. Learner Learn

如果再考虑多数派要 Learn 成功，这其实就是 Zab 协议。Paxos 算法着重是强调了选举过程的控制，对决议学习考虑的不多，Zab 恰好对此进行了补充。
