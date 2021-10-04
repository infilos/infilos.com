---
type: docs
title: "3PC"
linkTitle: "3PC"
weight: 9
---

三阶段提交协议最关键要解决的问题就是 Coordinator 和参与者同时挂掉导致数据不一致的问题，所以 3PC 在 2PC 的基础上又添加了一个阶段：CanCommit、PreCommit、DoCommit。

## 过程

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20190224210148.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### 1：CanCommit

1. 事务轮序：Coordinator 向各个参与者发送 CanCommit 请求，询问是否可以执行事务提交操作，并开始等待所有参与者的响应。
2. 参与者向 Coordinator 反馈询问的响应：参与者收到 CanCommit 请求之后，正常情况下，如果自身认为可以顺利执行事务，那么会返回 Yes 响应并进入预备专题，否则返回 No。

### 2：PreCommit

**执行事务预提交**：如果 Coordinator 接收到各个参与者的反馈都是 Yes，那么执行事务预提交：

1. 发送预提交请求：Coordinator 向各参与者发送 PreCommit 请求，进入 prepared 阶段。
2. 事务预提交：参与者接收到 PreCommit 请求之后，会执行事务操作，并将 Undo 和 Redo 信息记录到事务日志中。
3. 参与者向 Coordinator 反馈事务执行的响应：如果各参与者都成功执行了事务操作，那么反馈给 Coordinator ACK 响应，同时开始等待最终指令：提交 commit 或终止 abort，结束流程。

**中断事务**：如果任何一个参与者向 Coordinator 反馈了 No 响应，或者在等待超时后，Coordinator 无法接收到所有参与者的反馈，那么就会中断事务。

1. 发送中断请求：Coordinator 向所有参与者发送 abort 请求。
2. 中断事务：无论收到来自 Coordinator 的 abort 请求，还是等待超时，参与者都中断事务。

### 3：DoCommit

**执行提交**：

1. 发送提交事务：假设 Coordinator 正常工作，接收到了所有参与者的 ACK，那么他将从预提交阶段进入提交阶段，并向所有参与者发送 DoCommit 请求。
2. 事务提交：参与者收到 DoCommit 请求之后，正式提交事务，并在完成事务提交之后释放占用的资源。
3. 反馈事务提交结果：参与者完成事务提交之后，向 Coordinator 发送 ACK。
4. 完成事务：Coordinator 接收到所有参与者的 ACK，完成事务。

**中断事务**：假设 Coordinator 正常工作，并且有任一参与者反馈 No，或者在等待超时后无法接受到所有参与者的反馈，都会中断事务：

1. 发送中断请求：Coordinator 向所有参与者节点发送 abort 请求。
2. 事务回滚：参与者收到 abort 请求之后，利用 undo 日志执行事务回滚，并在完成事务回滚后释放占用资源。
3. 返回事务回滚结果：参与者在完成事务回滚之后，向 Coordinator 发送 ACK。
4. 中断事务：Coordinator 接收到所有参与者反馈的 ACK，中断事务。

## 分析

3PC 虽然解决了 Coordinator 与参与者均异常情况下导致数据不一致的问题，3PC 依然带来了其他问题。比如，网络分区问题，在 PreCommit 消息发出后突然两个机房断开网络，这时 Coordinator 所在机会会 abort，另外剩余的参与者的机房则会 commit。

而且由于 3PC 的设计过于复杂，在解决 2PC 问题的时候也引入了新的问题，因此实际应用并不广泛。

## Reference

- [分布式系统的一致性协议之 2PC 和 3PC](http://matt33.com/2018/07/08/distribute-system-consistency-protocol/)