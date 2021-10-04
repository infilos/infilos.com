---
type: docs
title: "BASE"
linkTitle: "BASE"
weight: 4
---

弱一致性BASE：

多数情况下，其实我们也并非一定要求强一致性，部分业务可以容忍一定程度的延迟一致，所以为了兼顾效率，发展出来了最终一致性理论BASE，BASE是指基本可用（Basically Available）、软状态（ Soft State）、最终一致性（ Eventual Consistency）

- 基本可用(Basically Available)：基本可用是指分布式系统在出现故障的时候，允许损失部分可用性，即保证核心可用。
- 软状态(Soft State)：软状态是指允许系统存在中间状态，而该中间状态不会影响系统整体可用性。分布式存储中一般一份数据至少会有三个副本，允许不同节点间副本同步的延时就是软状态的体现。
- 最终一致性(Eventual Consistency)：最终一致性是指系统中的所有数据副本经过一定时间后，最终能够达到一致的状态。弱一致性和强一致性相反，最终一致性是弱一致性的一种特殊情况。