---
title: MySQL undo log
date: 2019-05-09 11:15:21
tags: myql
---
# Undo log
`Undo log` 是 `InnoDB MVCC` 事务特性的重要组成部分。
`Undo log` 中保存用于实现 InnoDB 中的事务回滚；
对记录做了变更操作时就会产生 `undo` 记录，`Undo` 记录默认被记录到系统表空间(`ibdata`)中，但从5.6开始，也可以使用独立的 `Undo` 表空间。

## 何时记录 Undo log

* INSERT
在事务提交前只对当前事务可见，因此产生的Undo日志可以在事务提交后直接删除
* DELETE \ UPDATE
需要维护多版本信息
在 InnoDB 里，UPDATE 和 DELETE 操作产生的 Undo日志 被归成一类，即 `update_undo`

数据修改时，需要同时记录 redo log 和 undo log；如果因为某些原因导致事务失败或回滚了，可以借助该undo进行回滚；
`undo log` 和 `redo log` 记录物理日志不一样，它是逻辑日志。
可以认为：
1. 当 delete 一条记录时，`undo log` 中会记录一条对应的insert记录，反之亦然；
2. 当 update 一条记录时，它记录一条对应**相反**的 update 记录。

## Undo log 中记录什么
> 存储了老版本的数据，可能包含多个版本的历史数据；

当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着undo链找到满足其可见性的记录

# 基本文件结构
**rseg**（ rollback segment ）: 回滚段

* rseg0：预留在系统表空间ibdata中;
* rseg1 - rseg32：这32个回滚段存放于临时表的系统表空间中;
* rseg33 - rseg127: 则根据配置存放到独立undo表空间中（如果没有打开独立Undo表空间，则存放于ibdata中）

## 如何实现并发写入

## InnoDB 最大支持多少个普通事务