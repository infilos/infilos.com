---
type: docs
title: "Raft"
linkTitle: "Raft"
weight: 12
---

## 基本概念

### 复制状态机

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190224225452.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- 复制状态机通过日志复制来实现：
  - 日志：每台机器保存一份日志，日志来自客户端的请求，包含一系列命令。
  - 状态机：状态机会按序执行这些命令。
  - 一致性模型：分布式环境中，保证多机的日志是一致的，这样回放到状态机中得到的状态就是一致的。
- 一致性算法用于一致性模型，一般有以下特性：
  - Safety：在非拜占庭问题下(网络延时、网络分区、丢包、重复包、包乱序)，结果是正确的。
  - Availability：在半数以上机器能正常工作时，服务可用。
  - Timing-unindepentent：不依赖于时钟来保持日志一致性，错误的时钟以及极端的消息延时最多会造成可用性问题。

> 实际的实现中，建议状态机的每个命令曹邹都是幂等的，这样更易于保证一致性。
> 

### 服务器状态

每台服务器一定会处于三种状态：

- 领导者
- 候选者
- 追随者

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190224230043.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- 追随者只响应其他服务器的请求。如果追随者没有收到任何消息，它会成为一个候选者并开始一次选举。
- 收到大多数服务器投票的候选者将称为新的领导者。
- 领导者在宕机之前会一致保持领导者的状态。

### 任期

Raft 算法将事件划分为任意不同长度的任期(Term)。任期用连续的数字来表示。每个任期的开始都是一次选举(election)，一个或多个候选者会试图称为领导者。如果一个候选者赢得了选举，他就会在该任期的剩余时间内担任领导者。在某些情况下，选票会被瓜分，有可能没有选出领导者。这时将开始另一个任期，并且立刻开始下一次选举。Raft 算法保证在指定的任期内只有一个领导者。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190224230534.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### RPC

Raft 算法中服务器节点之间通过 RPC 通信，并且基本的一致性算法只需要两种类型的 RPC。请求投票 RPC 由候选者在选举期间发起，然后附加条目 RPC 由领导者发起，用来复制日志或提供一种心跳机制。为了服务器之间传输快照增加了第三种 RPC。当服务器没有及时收到 RPC 的响应时会尝试重试，并且它们能够并行的发起 RPC 来获得最佳性能。RPC 有三种：

1. RequestVote RPC：候选者在选举期间发起。
2. AppendEntries RPC：领导者发起的一种心跳机制，或用于日志复制。
3. InstallSnapshot RPC：领导者使用该 RPC 来发送快照给过于落后的追随者。

超时设置：

1. BroadcastTime：领导者的心跳超时。
2. ElectionTimeout：追随者设置的候选超时时间。
3. MTBF：指的是单个服务器发生故障的间隔时间的平均值。

BroadcastTime < ElectionTimeout < MTBT 原则：

1. BroadcastTime 应该比 ElectionTimeout 小一个数量级，为的是使领导者能够持续发送心跳信息来避免追随者开始发起选举。
2. ElectionTimeout 应该比 MTBT 小几个数量级，为的是使系统稳定运行。

一般 BroadcastTime 大约为 0.5 毫秒到 20 毫秒，ElectionTimeout 一般在 10ms 到 500ms 之间。大多数服务器的 MTBF 都在几个月甚至更长。

## 选举

**触发条件**：

1. 一般情况下，追随者接收到领导者的心跳时，会重置 ElectionTimeout，不会触发。
2. 领导者故障，追随者的 ElectionTimeout 发生超时，会转换为候选者，触发选举。

**候选操作过程**：

追随者自增当前任期，转换为候选者，对自己投票，并发起 RequestVote RPC，等待以下三种情况发生：

- 获得超过半数服务器的投票，赢得选举，称为领导者。
- 另一台服务器赢得选举，并接收到对应的心跳，称为追随者。
- 选举超时，没有任何一台服务器赢得选举，自增当前任期，重新发起选举。

**注意事项**：

- 服务器在一个任期内，最多只能给一个候选者投票，采用先到先服务原则。
- 候选者等待投票期间，可能会接收到来自其他声明称为领导者的 AppendEntries RPC。如果该领导人的任期(RCP 的内容)比当当前候选者的任期要大，则当前候选者认为该领导者合法，并转换称为追随者；如果 RPC 中的任期小于当前任期，则后选择拒绝此次 RPC，继续保持候选者状态。
- 候选者既没有赢得选举也没有输得选举：如果很多追随者在同一时刻都称为了候选者，选票会被分散，可能没有候选者获得较多的投票。当这种情况发生时，每一个候选者都会超时，并且通过自增任期号和发起另一轮 RequestVote RPC 来开始新的选举。然而，如果没有其他手段来分配选票的话，这种情况可能会无限制的重复下去。所以 Raft 使用的随机方式来设置选举超时时间(150~300ms)来避免这种情况的发生。

**问题探讨**：

- 候选者已经给自己投票了，一个候选者在一个任期内只会给一个人投票。
- 也有可能算法本身设定候选者就拒绝所有其他服务器的请求。

## 日志复制

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190224232428.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

**接收命令过程**：

1. 领导者接受客户端请求。
2. 领导者将命令追加到日志。
3. 发送 AppendEntries RPC 请求到追随者。
4. 领导者收到大多数追随者的确认后，领导者 Commit 日志，将日志在状态机中回放，并返回结果给客户端。


**提交过程**：

1. 在下一个心跳阶段，领导者再次发送 AppendEntries RPC 给追随者，日志已经 Commit 完成。
2. 追随者收到 Commit 结果之后，将日志在状态机中回放。

## 安全性

到目前为止描述的机制并不能充分的保证每一个状态机会按照相同的顺序执行相同的指令，例如：一个跟随者可能会进入不可用状态同时领导人已经提交了若干的日志条目，然后这个跟随者可能会被选举为领导人并且覆盖这些日志条目；因此，不同的状态机可能会执行不同的指令序列。

### 1. 领导者追加日志

领导者永远不会覆盖已经存在的日志条目；日志永远只有一个流向：从领导者到追随者；

### 2. 选举限制：投票阻止没有全部日志条目的服务器赢得选举

如果投票者的日志比候选人的新，拒绝投票请求；这意味着要赢得选举，候选者的日志至少和大多数服务器的日志一样新，那么它一定包含全部的已经提交的日志条目。

### 3. 永远不提交任期之前的日志条目（只提交任期内的日志条目）

在Raft算法中，当一个日志被安全的复制到绝大多数的机器上面，即AppendEntries RPC在绝大多数服务器正确返回了，那么这个日志就是被提交了，然后领导者会更新 “commit index”。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190224232951.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

如果允许提交任期之前的日志条目，那么在步骤 c 中，我们就会把之前任期为 2 的日志提交到其他服务器中去，并造成了大多数机器存在了日志为 2 的情况。所以造成了 d 中 S5 中任期为 3 的日志条目会覆盖掉已经提交的日志的情况。

Raft 从来不会通过计算复制的数目来提交之前人气的日志条目。只有领导人当前任期的日志条目才能通过计算数目来进行提交。一旦当前任期的日志条目以这种方式被提交，那么由于日志匹配原则（Log Matching Property），之前的日志条目也都会被间接的提交。

论文中的这段话比较难理解，更加直观的说：由于 Raft 不会提交任期之前的日志条目，那么就不会从 b 过渡到 c 的情况，只能从 b 发生 S5 宕机的情况下直接过渡到 e，这样就产生的更新的任期，这样 S5 就没有机会被选为领导者了。

### 4. 候选者和追随者崩溃

候选者和追随者崩溃的情况处理要简单的多。如果这类角色崩溃了，那么后续发送给他们的 RequestVote 和 AppendEntries 的所有 RPC 都会失败，Raft 算法中处理这类失败就是简单的无限重试的方式。如果这些服务器重新可用，那么这些 RPC 就会成功返回。如果一个服务器完成了一个 RPC，但是在响应 Leader 前崩溃了，那么当他再次可用的时候还会收到相同的 RPC 请求，此时接收服务器负责检查，比如如果收到了已经包含该条日志的 RPC 请求，可以直接忽略这个请求，确保对系统是无害的。

## 集群成员变更

集群成员的变更和成员的宕机与重启不同，因为前者会修改成员个数进而影响到领导者的选取和决议过程，因为在分布式系统这对于 majority 这个集群中成员大多数的概念是极为重要的。

简单的做法是，运维人员将系统临时下线，修改配置，重新上线。但是这种做法存在两个缺点：

- 更改时集群不可用
- 认为操作失误风险

### 直接从一种配置转到新的配置是十分不安全的

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190224233534.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

因为各个机器可能在任何的时候进行转换。在这个例子中，集群配额从 3 台机器变成了 5 台。不幸的是，存在这样的一个时间点，两个不同的领导人在同一个任期里都可以被选举成功。一个是通过旧的配置，一个通过新的配置。

### 两阶段方法保证安全性

为了保证安全性，配置更改必须使用两阶段方法。在 Raft 中，集群先切换到一个过渡的配置，我们称之为共同一致；一旦共同一致已经被提交了，那么系统就切换到新的配置上。共同一致是老配置和新配置的结合。

共同一致允许独立的服务器在不影响安全性的前提下，在不同的时间进行配置转换过程。此外，共同一致可以让集群在配置转换的过程人依然响应服务器请求。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190224233844.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

一个领导人接收到一个改变配置从 C-old 到 C-new 的请求，他会为了共同一致存储配置（图中的 C-old,new），以前面描述的日志条目和副本的形式。一旦一个服务器将新的配置日志条目增加到它的日志中，他就会用这个配置来做出未来所有的决定。领导人完全特性保证了只有拥有 C-old,new 日志条目的服务器才有可能被选举为领导人。当 C-old,new 日志条目被提交以后，领导人在使用相同的策略提交 C-new，如下图所示，C-old 和 C-new 没有任何机会同时做出单方面的决定，这就保证了安全性。

上图是一个配置切换的时间线。虚线表示已经被创建但是还没有被提交的条目，实线表示最后被提交的日志条目。领导人首先创建了 C-old,new 的配置条目在自己的日志中，并提交到 C-old,new 中（C-old,new 的大多数和 C-new 的大多数）。然后他创建 C-new 条目并提交到 C-new 中的大多数。这样就不存在 C-new 和 C-old 可以同时做出决定的时间点。

## 日志压缩

日志会随着系统的不断运行会无限制的增长，这会给存储带来压力，几乎所有的分布式系统(Chubby、ZooKeeper)都采用快照的方式进行日志压缩，做完快照之后快照会在稳定持久存储中保存，而快照之前的日志和快照就可以丢弃掉。

Raft的具体做法如下图所示：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190224234023.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

与 Raft 其它操作 Leader-Based 不同，snapshot 是由各个节点独立生成的。除了日志压缩这一个作用之外，snapshot 还可以用于同步状态：slow-follower 以及 new-server，Raft 使用 InstallSnapshot RPC 完成该过程，不再赘述。

## Client 交互

- Client 只向领导者发送请求；
- Client 开始会向追随者发送请求，追随者拒绝 Client 的请求，并重定向到领导者；
- Client 请求失败，会超时重新发送请求。

Raft 算法要求 Client 的请求线性化，防止请求被多次执行。有两个解决方案：

- Raft 算法提出要求每个请求有个唯一标识；
- Raft 的请求保持幂等性。

## Reference

- [论文](http://blog.luoyuanhang.com/2017/02/02/raft-paper-in-zh-CN/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
- ETCD Raft 库的实现
- [TiKV 源码解析系列 - Raft 的优化](https://mp.weixin.qq.com/s?__biz=MzI3NDIxNTQyOQ==&mid=2247484544&idx=1&sn=7d8e412ecc5aaeb3f9b7cf391bdcf398&chksm=eb1623eadc61aafcefcfbdf36b388a5f96d3009d21641eb6ac67c57317d6c397ddeb58fc7d06#rd)
- [Elasticell-Multi-Raft实现](https://zhuanlan.zhihu.com/p/33047950?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
- [Raft 理解](http://tinylcy.me/2018/Understanding-the-Raft-consensus-algorithm-Two/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
- [Raft 一致性算法笔记](https://juejin.im/entry/59bf3643f265da0655052fba)
- [Raft 协议理解](https://juejin.im/post/5aed9a7551882506a36c659e)
- [Raft 算法详解](https://zhuanlan.zhihu.com/p/32052223)