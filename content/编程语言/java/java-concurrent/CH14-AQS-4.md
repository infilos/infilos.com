---
type: docs
title: "CH14-AQS-4"
linkTitle: "CH14-AQS-4"
weight: 14
---

## AQS 总结

最核心的就是sync queue的分析。

- 每个节点都是由前驱节点唤醒。
- 如果节点发现前驱节点是 head 并且尝试获取成功，则会轮到该线程执行。
- condition queue 中的节点想 sync queue 中转移是通过 signal 操作完成的。
- 当节点状态为 SIGNAL 时，表示后面的节点需要运行。

