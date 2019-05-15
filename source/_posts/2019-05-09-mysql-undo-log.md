---
title: MySQL undo log
date: 2019-05-09 11:15:21
tags: myql
---

日志文件主要用于记录redo log，InnoDB采用循环使用的方式，你可以通过参数指定创建文件的个数和每个文件的大小。默认情况下，日志是以512字节的block单位写入。由于现代文件系统的block size通常设置到4k，InnoDB提供了一个选项，可以让用户将写入的redo日志填充到4KB，以避免read-modify-write的现象；而Percona Server则提供了另外一个选项，支持直接将redo日志的block size修改成指定的值。


从上层的角度来看，InnoDB层的文件，除了redo日志外，基本上具有相当统一的结构，都是固定block大小，普遍使用的btree结构来管理数据。只是针对不同的block的应用场景会分配不同的页类型。通常默认情况下，每个block的大小为 UNIV_PAGE_SIZE，在不做任何配置时值为16kb

作用：
　　保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读
内容：
　　逻辑格式的日志，在执行undo的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，而不是从物理页面上操作实现的，这一点是不同于redo log的。
什么时候产生：
　　事务开始之前，将当前是的版本生成undo log，undo 也会产生 redo 来保证undo log的可靠性
什么时候释放：
　　当事务提交之后，undo log并不能立马被删除，
　　而是放入待清理的链表，由purge线程判断是否由其他事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间。

查询表空间使用情况
> show variables like '%innodb_data_file_path%'

# Undo log (回滚日志)


`Undo log` 是 `InnoDB MVCC` 事务特性的重要组成部分。
`Undo log` 中保存用于实现 InnoDB 中的事务回滚；
对记录做了变更操作时就会产生 `undo` 记录，`Undo` 记录默认被记录到系统表空间(`ibdata`)中，但从5.6开始，也可以使用独立的 `Undo` 表空间。
`Undo log` = `undo log records`(对应一个事务) * N

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
> An undo log record contains information about how to undo the latest change by a transaction to a clustered index record.

当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着undo链找到满足其可见性的记录

## 关联
* 128 个 rollback segment
* 一个 rollback segment 有 1024 个 undo slot；
* 一个 undo slot 对应一个 undo log
* 一个 事务(dml) 对应一个 undo log

## InnoDB 最大支持多少个普通事务（user defined）
- 一个事务仅含 insert 或 update/delete
> **( innodb_page_size / 16 ) \* *innodb_rollback_segments* \* number of undo tablespaces**
- 一个事务 含 一个insert 或 一个update\delete
> **( innodb_page_size / 16 / 2 ) \* innodb_rollback_segments \* number of undo tablespaces**
- 一个事务在临时表空间执行 insert
> **( innodb_page_size / 16 ) \* innodb_rollback_segments**
- 一个事务在临时表空间 执行insert 且 update\delete
> **( innodb_page_size / 16 / 2 ) \* innodb_rollback_segments**

### 支持事务数量调整
[5.5 支持事务数量](https://dev.mysql.com/doc/refman/5.5/en/innodb-undo-logs.html)：5.1+版本，`1024 * innodb_rollback_segments`
[8.0 支持事务数量](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-logs.html)：5.6+版本，增加配置参数 `number of undo tablespaces`
> In MySQL 5.6 and MySQL 5.7, the number of undo tablespaces is controlled by the innodb_undo_tablespaces configuration option. In MySQL 8.0, two default undo tablespaces are created when the MySQL instance is initialized, and additional undo tablespaces can be created using CREATE UNDO TABLESPACE syntax.

# 基本文件结构
undo tablesapce \ global temporary tablespace (used for transactions that modify data in user-defined temporary tables)
--> rollback segment
----> undo log segment
------> undo log
--------> undo log record

**rseg**（ rollback segment ）: 回滚段

* rseg0：预留在系统表空间ibdata中;
* rseg1 - rseg32：这32个回滚段存放于临时表的系统表空间中;
* rseg33 - rseg127: 则根据配置存放到独立undo表空间中（如果没有打开独立Undo表空间，则存放于ibdata中）

## 如何实现并发写入

# 执行流程
## 创建 undo 物理表空间
> srv_undo_tablespaces_init

1. 根据 `innodb_undo_tablespaces` 参数，调用 `srv_undo_tablespace_create` 创建默认大小为 10M 的文件；
- 分别对 `undo tablespace` 调用 `srv_undo_tablespace_open`；主要调用`fil_space_create` 和 `fil_node_create` 将新建立的 `undo tablespace` 加入 Innodb 的文件体系；
- 对 `undo tablespace` 进行 fsp_header 的初始化
```
    mlog_write_ulint(header + FSP_SPACE_ID, space_id, MLOG_4BYTES, mtr);
    mlog_write_ulint(header + FSP_NOT_USED, 0, MLOG_4BYTES, mtr);

    mlog_write_ulint(header + FSP_SIZE, size, MLOG_4BYTES, mtr);
    mlog_write_ulint(header + FSP_FREE_LIMIT, 0, MLOG_4BYTES, mtr);
    mlog_write_ulint(header + FSP_SPACE_FLAGS, space->flags,
             MLOG_4BYTES, mtr);
    mlog_write_ulint(header + FSP_FRAG_N_USED, 0, MLOG_4BYTES, mtr);

    flst_init(header + FSP_FREE, mtr);
    flst_init(header + FSP_FREE_FRAG, mtr);
    flst_init(header + FSP_FULL_FRAG, mtr);
    flst_init(header + FSP_SEG_INODES_FULL, mtr);
    flst_init(header + FSP_SEG_INODES_FREE, mtr);
```
生成了 `n_undo_tablespaces` 个大小为10MB的 `undo tablespace` 文件，并且已经加入到 innoDB 文件体系，但是里面没有任何类容。

## ibdata中system segment header的初始化
调用 `trx_sys_create_sys_pages->trx_sysf_create` 进行，除初始化`transaction system segment` 外还会初始化其 `header( ibdata page no 5)` 信息；
老版本代码中，TRX_SYS_OLD_N_RSEGS=256，实际只会有128个，
rollback segment slots 都初始化完成(源码所示有256个，实际上最多只会有128个，其中0号solt固定在ibdata中)

slot 大小是TRX_SYS_RSEG_SLOT_SIZE设置的大小为8字节，4字节space id ,4字节 page no，它们会指向 rollback segment header所在的位置
TRX_SYS_TRX_ID_STORE
TRX_SYS_FSEG_HEADER
TRX_SYS_RSEGS

## rollback segment header 初始化
调用 trx_sys_create_rsegs进行；

关于innodb_undo_logs参数和innodb_rollback_segments参数，他们作用就是设置rollback segment 的个数，本文以128为例。

根据注释和代码innodb_undo_logs已经是个淘汰的参数，应该用innodb_rollback_segments代替。
这两个参数默认是就是TRX_SYS_N_RSEGS及 128 其实不用设置的

1. 初始化rollback segments 段
```
n_noredo_created = trx_sys_create_noredo_rsegs(n_tmp_rsegs); 
//创建 32个 临时rollback segments
```
2. 建立 95个(33-128) 普通rollback segments
是 `i % n_spaces` 的取模方式 `n_spaces` 为 `innodb_undo_tablespaces` 参数设置的值，因此每个 `rollback segment` 是轮序的方式分布到不同的 `undo tablespace` 中的。

3. rollback segment header初始化过程
建立rollback segment
```
block = fseg_create(space, 0, TRX_RSEG + TRX_RSEG_FSEG_HEADER, mtr); 
//建立一个回滚段，返回段头所在的块
```
初始化TRX_RSEG_MAX_SIZE和TRX_RSEG_HISTORY_SIZE信息
初始化每个undo segment header所在的page no；




# 相关参数
## innodb_rollback_segments
> 每个 `undo tablespace` 和 `global temporary tablespace` 分别支持 128 个`rollback segment`
> Each undo tablespace and the global temporary tablespace individually support a maximum of 128 rollback segments. The innodb_rollback_segments variable defines the number of rollback segments.

## innodb_undo_tablespaces
[innodb_undo_tablespaces](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_undo_tablespaces)

## innodb_page_size

# 问题
## 什么是 purge
1 delete语句操作的后，只会对其进行delete mark，这些被标记为删除的记录只能通过purge来进行物理的删除，但是并不回收空间
2 undo log，如果undo 没有任何事务再引用，那么也只能通过purge线程来进行物理的删除，但是并不回收空间

## purge后空间就释放了吗
1 undo page里面可以存放多个undo log日志
2 只有当undo page里面的所有undo log日志都被purge掉之后，这个页的空间才可能被释放掉，否则这些undo page可以被重用