---
type: docs
title: "CRDT"
linkTitle: "CRDT"
weight: 7
---

我们了解到分布式一些规律原则之后，就要着手考虑如何来实现解决方案，一致性算法的前提是数据结构，或者说一切算法的根基都是数据结构，设计良好的数据结构加上精妙的算法可以高效的解决现实的问题。经过前人不断的探索，我们得知分布式系统被广泛采用的数据结构CRDT。

参考《谈谈CRDT》,A comprehensive study of Convergent and Commutative Replicated Data Types

- 基于状态(state-based)：即将各个节点之间的CRDT数据直接进行合并，所有节点都能最终合并到同一个状态，数据合并的顺序不会影响到最终的结果。
- 基于操作(operation-based)：将每一次对数据的操作通知给其他节点。只要节点知道了对数据的所有操作（收到操作的顺序可以是任意的），就能合并到同一个状态。

----

## 什么是 CRDT

CRDT 是 Conflict-Free Replicated Data Type 的缩写，即**无冲突的可复制数据类型**。

它用于解决分布式系统的最终一致性问题，即，在分布式系统中，应用采用什么样的数据结构来保证最终一致性？而 CRDT 是目前理论界给出的答案，相关论文为 [A comprehensive study of Convergent and Commutative Replicated Data Types](http://hal.upmc.fr/file/index/docid/555588/filename/techreport.pdf)。

## 一致性的难题

构建一个分布式系统并不难，而难的是构建一个与单机系统的正确性一样的分布式系统，即 [CAP 定理](https://en.wikipedia.org/wiki/CAP_theorem)。

CAP 定理告诉我们，在构建分布式系统时，Consistency(一致性)、Availability(可用性)、Partition tolerance(分区容错性)，三者只能同时选取两项。

其中，分区容错性是任何生产环境下的分布式系统所必须的，因此，只有在 C、A 之间做出取舍：

- 选择一致性：构建一个强一致性系统，比如符合 ACID 特性的数据库系统。
- 选择可用性：构建一个最终一致性系统，比如 NoSQL 系统。

选择一致性时数据一旦落地就是一致的，但是可用性不能实时保证，比如系统有时忙于一致性处理，无法对外提供服务。

选择可用性时则时刻都能保证可用，但是各个节点在同一时刻所持有的数据可能并不一致，但经过一段时间后，数据在各节点间会达到一致状态。

**因此，现在的分布式系统总是会偏向于选择 AP，以提供一个无单点故障、总是可用且更高吞吐的系统。**

## 使用那些信息能够达到最终一致

在实际应用中，我们需要考虑多种数据类型的应用和场景，设计一个能够保证最终一致性的数据结构会变得很复杂。

而 CRDT 就是这样一些适用于不同场景的、可以保持最终一致性的数据结构的统称。围绕 CRDT 理论，则涵盖了：

- 它们应该具有怎样的基本表现形式
- 满足一些什么样的条件才可以保持最终一致性，毕竟不能每次都穷举所有情况
- 不断寻找一些通用的、有大量应用场景的 CRDT，并努力提高其空间、时间效率

前面提到的 CRDT 相关论文总结了目前为止人们在 CRDT 这件事情上的认识程度，简要总结如下：

- 定义了 CRDT
- 列举了 CRDT 的两种基本形式：
    - 基于状态的 CRDT：存储最终值
    - 基于操作的 CRDT：存储操作记录
- 界定了 CRDT 能够满足最终一致性的边界条件。比如，设计一个 CRDT，只需要验证它是否满足这些边界条件，即可知道它是否能够保持最终一致性
- 界定了两类 CRDT 在系统中应用时，需要的信息交换的边界条件。即回答怎样才能叫做”收集到足够多的信息“
- 枚举了当前已知的 CRDT，包括计数器(counter)、寄存器(register)、集合(set)、图(graph)等几个种类
- 在现实中应用如何应用 CRDT，尤其是如何回收存储空间的问题

## 如何在实际系统中应用

最终一致性分布式框架 RiakCore 的应用方式：

- 抛弃自己缩写的数据结构，实现 CRDT，或者使用已有的 CRDT Library
- 参考 CRDT 的一致性可判断条件(即”收集到足够多的信息“)，在需要判断最终一致性时收集它们
- 抛弃自己所写的一致性判断算法，实现 CRDT 的一致性合并算法，或者使用已有的 CRDT Library

> [Riak](https://github.com/basho/riak) 是一个由 Erlang 编写的分布式、去中心化的数据存储系统，[Riak Core](https://github.com/basho/riak_core) 定义了其数据分发和扩展的形式，可以被认为是一个用于构建分布式、可扩展、高容错应用的工具集。

## 谁应该使用？

所有力求追踪一致性的系统，都应该使用 CRDT。如果一个最终一致性的分布式系统还有没有使用 CRDT，要么是其所使用的数据结构已经实现了 CRDT 的一种或几种，虽然可能很粗糙，要么是这个系统在最终一致性上的保证存在问题。

## 总结

CRDT 并未给用户层面带来影响，但是从管理员、开发者的角度来看，CRDT 给了我们基于逻辑来判断分布式系统能否保证最终一致性的能力。

## Reference

- [Eventual Consistency：谈谈 CRDT](http://blog.chinaunix.net/uid-608135-id-4730055.html)
- [http://liyu1981.github.io/what-is-CRDT/](http://liyu1981.github.io/what-is-CRDT/)
- [http://christophermeiklejohn.com/crdt/2014/07/22/readings-in-crdts.html](http://christophermeiklejohn.com/crdt/2014/07/22/readings-in-crdts.html)

