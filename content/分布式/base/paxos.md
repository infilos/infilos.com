---
type: docs
title: "Paxos"
linkTitle: "Paxos"
weight: 10
---

分布式系统除了能提升整个系统的性能外还有一个重要的特性就是提高系统的可靠性，可靠性指的是当分布式系统中一台或N台机器宕掉后都不会导致系统不可用，分布式系统是 “state machine replication” 的，每个节点都可能是其他节点的快照，这是保证分布式系统高可靠性的关键，而存在多个复制节点就会存在数据不一致的问题，这时一致性就成了分布式系统的核心；在分布式系统中必须保证：

**加入在分布式系统中初始时各个节点的数据是一致的，每个节点都顺序执行系列操作，然后每个节点最终的数据还是一致的。**

**一致性算法**：用于保证在分布式系统中每个节点都顺序执行相同的操作序列，在每个指令上执行**一致性算法**就能保证各个节点最终的数据是一致的。

Paxos 就是用于解决一致性问题的算法。有多个节点就会存在节点之间的通信问题，有两种通信模型：共享内存、消息传递。Paxos 是基于消息传递的通信模型。

## 概述

> Paxos 用于解决分布式系统中的一致性问题。在一个 Paxos 过程只批准一个 Value，只有被 Prepare 的 Value 且被多数 Acceptor 接受才能被批准，被批准的 Value 只能被 Learner。
> 

流程简述：

1. 有一个 Client、三个 Proposer、三个 Acceptor、一个 Leaner；
2. Client 向 Prepare 提交一个 Data 请求入库，Proposer 接收到 Client 请求后生成一个序列号 1 向三个 Acceptor(或最少两个)发送序号 1 请求提案。
3. 假如三个 Acceptor 收到 Proposer 申请提交的序号 1 提案，且三个 Acceptor 都是初次接收到该提案，这时向 Proposer 回复 Promise 允许提交的提案。
4. Proposer 收到三个 Acceptor(满足过半原则)的 Promise 回复后接着向三个 Acceptor 正式提交提案(序号 1，value 为 data)。
5. 三个 Acceptor 都受到该提案，请求期间没有收到其他请求，Acceptor 则接受提案，回复 Proposer 已接受提案，然后向 Learner 提交提案。
6. Proposer 收到回复后给 Client 成功处理请求。
7. Learner 收到提案后开始学习提案(存储 Data)。

角色划分：

- Proposer：提议者
- Acceptor：决策者
- Learner：提案学习者

阶段划分：

1. 准备阶段
    1. Proposer 向超过半数(n/2+1)的 Acceptor 发起 Prepare 消息(提案编号)。
    2. 如果 Prepare 符合协议规则，Acceptor 回复 Promise 消息，否则拒绝。
2. 决议阶段(投票阶段)
    1. 如果超过半数 Acceptor 回复 Promise，Proposer 向 Acceptor 发送 Accept 消息。
    2. Acceptor 检查 Accept 消息是否符合规则，消息符合规则则批准 Accept 请求。

## 详解

### Paxos 保证

- 只有提出的议案才能被选中，没有议案提出就不会被选中。
- 多个被提出的议案中只有一个议案会被选中。
- 议案被选中后 Learner 就可以开始学习该议案。

### 约束条件

**P1-Acceptor 必须接受它接收到的第一个议案**。有约束就会出现一个问题：当多个议案被多个 Proposer 同时提出，这时每个 Acceptor 都接收到了它们各自的第一个议案，此时无法选择最终议案。所以就需要另一个约束 P2。

**P2-一个议案被选中需要过半的 Acceptor 接受**。

假设 A 为整个 Acceptor 集合；B 为超过 A 一半的 Acceptor 集合，B 为 A 的子集；C 也是超过 A 半数的 Acceptor 集合，C 也是 A 的子集。由此可知，任意两个超过半数的子集中必定有一个相同的成员 Acceptor。

此说明了一个 Acceptor 可以接受不止一个议案，此时需要一个编号来标识每个议案，议案的编号格式为：(编号，Value)。编号为不可重复且全序。

因为一个 Paxos 过程只能批准一个 Value，这时退出了约束 P3。

**P3-当编号为 K0、Value 为 V0 的议案(K0,V0)被过半的 Acceptor 接受后，今后(同一个 Paxos 或称一个 Round 中)，所有比 K0 更高编号且被 Acceptor 接受的议案，其 Value 必须为 V0**。

因为每个 Proposer 都可以提出多个议案，每个议案最初都有一个不同的 Value，所有要满足 P3 就又要退出一个新的约束 P4。

**P4-只有 Acceptor 没有接受过议案，Proposer 才能采用自己的 Value，否则 Proposer 的 Value 议案为 Acceptor 中编号最大的 Proposer Value**。

### Paxos 流程

这里具体例子来说明 Paxos 的整个具体流程： 假如有 Server1、Server2、Server3 这样三台服务器，我们要从中选出 leader，这时候 Paxos 派上用场了。整个选举的结构图如下：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190224215847.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

#### 1-准备阶段

1. 每个 Server 都向 Proposer 发消息称自己想成为 leader，Server1 往 Proposer1 发、Server2 往 Proposer2 发、Server3 往 Proposer3 发；
2. 现在每个 Proposer 都接收到了 Server1 发来的消息但时间不一样， Proposer2 先接收到了，然后是 Proposer1，接着才是 Proposer3；
    1. Proposer2 首先接收到消息所以他从系统中取得一个编号 1，Proposer2 向 Acceptor2 和 Acceptor3 发送一条，编号为 1 的消息；
    2. 接着 Proposer1 也接收到了 Server1 发来的消息，取得一个编号 2，Proposer1 向 Acceptor1 和 Acceptor2 发送一条，编号为 2 的消息；
    3. 最后 Proposer3 也接收到了 Server3 发来的消息，取得一个编号 3，Proposer3 向 Acceptor2 和 Acceptor3 发送一条，编号为 3 的消息；
3. 这时 Proposer1 发送的消息先到达 Acceptor1 和 Acceptor2，这两个都没有接收过请求所以接受了请求返回 (2,null) 给 Proposer1，并承诺不接受编号小于 2 的请求；
4. 此时 Proposer2 发送的消息到达 Acceptor2 和 Acceptor3，Acceprot3 没有接收过请求则返回 (1,null) 给 Proposer2，并承诺不接受编号小于 1 的请求，但这时 Acceptor2 已经接受过 Proposer1 的请求并承诺不接受编号小于的 2 的请求了，所以 Acceptor2 拒绝 Proposer2 的请求；
5. 最后 Proposer3 发送的消息到达 Acceptor2 和 Acceptor3， Acceptor2 接受过提议，但此时编号为 3 大于 Acceptor2 的承诺 2 与 Accetpor3 的承诺 1，所以接受提议返回 (3,null);
6. Proposer2 没收到过半的回复所以重新取得编号 4，并发送给 Acceptor2 和 Acceptor3，然后A cceptor2 和 Acceptor3 都收到消息，此时编号 4 大于 Acceptor2 与 Accetpor3 的承诺 3，所以接受提议返回 (4,null)；

#### 2-决议阶段

1. Proposer3 收到过半的返回，并且返回的 Value 为 null，所以Proposer3 提交了 (3,server3) 的议案；
2. Proposer1 收到过半返回，返回的 Value 为 null，所以 Proposer1提交了 (2,server1) 的议案；
3. Proposer2 收到过半返回，返回的 Value 为 null，所以 Proposer2 提交了 (4,server2) 的议案；
4. Acceptor1、Acceptor2 接收到 Proposer1 的提案 (2,server1) 请求，Acceptor2 承诺编号大于 4 所以拒绝了通过，Acceptor1 通过了请求；
5. Proposer2 的提案 (4,server2) 发送到了Acceptor2、Acceptor3，提案编号为 4 所以 Acceptor2、Acceptor3 都通过了提案请求；
6. Acceptor2、Acceptor3 接收到 Proposer3 的提案 (3,server3) 请求，Acceptor2、Acceptor3 承诺编号大于 4 所以拒绝了提案；
7. 此时过半的 Acceptor 都接受了 Proposer2 的提案 (4,server2)，Larner 感知到了提案的通过，Larner 学习提案，server2 成为 Leader；

一个 Paxos 过程只会产生一个议案所以至此这个流程结束，选举结果 Server2 为 Leader。

## Reference

- [Paxos Made Simple](https://github.com/oldratlee/translations/blob/master/paxos-made-simple/README.rst)
- [PaxosLease：实现租约的无盘Paxos算法](https://github.com/oldratlee/translations/blob/master/paxoslease/README.rst)
- [Paxos 过程](http://www.solinx.co/archives/403)
- [Paxos 图解](http://codemacro.com/2014/10/15/explain-poxos/)
- [Paxos 详解](https://lfwen.site/2016/12/25/paxos-algorithm/)