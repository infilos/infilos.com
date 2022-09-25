---
type: docs
title: "比较 DELETE/TRUNCATE/DROP"
linkTitle: "比较 DELETE/TRUNCATE/DROP"
weight: 1
---

## 相同点：

1. truncate 和不带 where 子句的 delete、drop 都会删除表内的数据。
2. drop、truncate 都是 DDL 语句(数据定义语言)，执行后会自动提交。

## 不同点：

1. 数据与结构：
   - truncate 和 delete 只删除数据不删除表的结构(定义)
   - drop 语句将删除表的结构被依赖的约束(constrain)、触发器(trigger)、索引(index)；依赖于该表的存储过程/函数将保留,但是变为 invalid 状态。
2. 事务：
   - delete 语句是数据库操作语言(dml)，这个操作会放到 rollback segement 中，事务提交之后才生效；如果有相应的 trigger，执行的时候将被触发。
   - truncate、drop 是数据库定义语言(ddl)，操作立即生效，原数据不放到 rollback segment 中，不能回滚，操作不触发 trigger。
3. 存储占用：
   - delete 语句不影响表所占用的 extent，高水线(high watermark)保持原位置不动
   - drop 语句将表所占用的空间全部释放。
   - truncate 语句缺省情况下见空间释放到 minextents个 extent，除非使用reuse storage；truncate 会将高水线复位(回到最开始)。
4. 速度：
   - 一般来说: drop > truncate > delete
5. 安全性：
   - 小心使用 drop 和 truncate，尤其没有备份的时候。
6. 应用场景：
   - 想删除部分数据行用 delete，注意带上where子句. 回滚段要足够大.
   - 想删除表,当然用 drop
   - 想保留表而将所有数据删除，如果和事务无关，用truncate即可。
   - 如果和事务有关,或者想触发trigger,还是用delete。
   - 如果是整理表内部的碎片，可以用truncate跟上reuse stroage，再重新导入/插入数据。