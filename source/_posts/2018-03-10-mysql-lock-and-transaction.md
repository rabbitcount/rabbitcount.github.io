---
title: mysql的那些事儿（锁、事务）
date: 2018-03-10 10:17:51
tags:
- mysql
- ACID
- 事务隔离级别
---

官方文档章节：[InnoDB Locking and Transaction Model](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-transaction-model.html) (Locking, Transaction Model, Locks Set by Different SQL Statements in InnoDB, Phantom Rows, Deadlocks in InnoDB)

- 锁类型(lock type)
- 事务隔离级别（Transaction isolation）和锁策略（locking strategis）；以及自动提交（autocommit），一致性非锁定读（consistent non-blocking），锁定读（locking read）
- “Locks Set by Different SQL Statements in InnoDB” discusses specific types of locks set in InnoDB for various statements. 
- Phantom rows（how InnoDB uses next-key locking to avoid phantom rows.）
- 死锁（deadlock）

# key word

共享锁（S）、排他锁（X）、意向共享（IS）、意向排他（IX）

1. Record Lock：单个行记录上的锁。
2. Gap Lock：间隙锁，锁定一个范围，但不包括记录本身。GAP锁的目的，是为了防止同一事务的两次当前读，出现幻读的情况。
3. Next-Key Lock：1 + 2，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题。

# Multi-Versioning

In the `InnoDB` transaction model, the goal is to combine the best properties of a [multi-versioning](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_mvcc "MVCC") database with traditional two-phase locking. 

# InnoDB的锁（locking）

## 共享锁(<font color=#0099ff>S</font>hard Locks)与排它锁(<font color=#0099ff>_E_</font>xclusive Locks)
**S锁(shared locks, S locks)** 和 **X锁(exclusive locks, X locks)** 同为 **行级锁（row-level locking）**
1. A shared (<font color=#0099ff>S</font>) lock permits the transaction that holds the lock to <font color=#0099ff>read a row</font>.
2. An exclusive(<font color=#0099ff>X</font>) lock permits the transaction that holds the lock to <font color=#0099ff>update or delete a row</font>.

> 多个线程可以同时持有S锁，X锁仅能被唯一线程持有（其他线程请求X锁，将会等待 wait to be granted）；

## Intention Locks

> Gap Locks中存在一种插入意向锁(Insert Intention Lock)，在INSERT操作时产生。

### ref

`InnoDB` supports _multiple granularity locking_ which permits coexistence of row locks and table locks. For example, a statement such as [`LOCK TABLES ... WRITE`](https://dev.mysql.com/doc/refman/5.7/en/lock-tables.html "13.3.5 LOCK TABLES and UNLOCK TABLES Syntax") takes an exclusive lock (an `X`
lock) on the specified table. To make locking at multiple granularity levels practical, `InnoDB` uses
[intention locks](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_intention_lock "intention lock").
Intention locks are table-level locks that indicate which type of lock (shared or exclusive) a transaction requires later for a row in a table. There are two types of intention locks:

* An [intention shared lock](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_intention_shared_lock "intention shared lock") (`IS`) indicates that a transaction intends to set a _shared_ lock on individual rows in a table.
* An [intention exclusive lock](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_intention_exclusive_lock "intention exclusive lock") (`IX`) indicates that that a transaction intends to set an exclusive lock on individual rows in a table.

For example, [`SELECT ... LOCK IN SHARE MODE`](https://dev.mysql.com/doc/refman/5.7/en/select.html "13.2.9 SELECT Syntax") sets an `IS` lock, and [`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/5.7/en/select.html "13.2.9 SELECT Syntax") sets an `IX` lock.

The intention locking protocol is as follows:

* Before a transaction can acquire a shared lock on a row in a table, it must first acquire an `IS` lock or stronger on the table.
* Before a transaction can acquire an exclusive lock on a row in a table, it must first acquire an `IX` lock on the table.

Table-level lock type compatibility is summarized in the following matrix.

|     X    |     IX     |     S      |    IS
------ | -------- | ---------- | ---------- | ----------
   X   | Conflict | Conflict   | Conflict   | Conflict  
   IX  | Conflict | Compatible | Conflict   | Compatible
   S   | Conflict | Conflict   | Compatible | Compatible
   IS  | Conflict | Compatible | Compatible | Compatible

A lock is granted to a requesting transaction if it is compatible with existing locks, but not if it conflicts with existing locks. A transaction waits until the conflicting existing lock is released. If a lock request conflicts with an existing lock and cannot be granted because it would cause [deadlock](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_deadlock "deadlock"), an error occurs.

Intention locks do not block anything except full table requests (for example, [`LOCK TABLES ... WRITE`](https://dev.mysql.com/doc/refman/5.7/en/lock-tables.html "13.3.5 LOCK TABLES and UNLOCK TABLES Syntax")). The main purpose of intention locks is to show that someone is locking a row, or going to lock a row in the table.

Transaction data for an intention lock appears similar to the following in [`SHOW ENGINE INNODB STATUS`](https://dev.mysql.com/doc/refman/5.7/en/show-engine.html "13.7.5.15 SHOW ENGINE Syntax") and [InnoDB monitor](https://dev.mysql.com/doc/refman/5.7/en/innodb-standard-monitor.html "14.17.3 InnoDB Standard Monitor and Lock Monitor Output")
output:

```
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```

## Record Locks

> - 通过 `for update` 占用 `record locks`，防止其他线程 **CUD**；
- `Record Locks` 创建在索引上，如果没有索引，会创建 `hidden clusterd index`
- 日志样式：`lock_mode X locks rec but not gap`

### ref

A record lock is a lock on an index record. For example, `SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;` prevents any other transaction from inserting, updating, or deleting rows where the value of `t.c1` is `10`.

Record locks always lock index records, even if a table is defined with no indexes. For such cases, `InnoDB` creates a hidden clustered index and uses this index for record locking. See [Section 14.8.2.1, “Clustered and Secondary Indexes”](https://dev.mysql.com/doc/refman/5.7/en/innodb-index-types.html "14.8.2.1 Clustered and Secondary Indexes").

Transaction data for a record lock appears similar to the following in [`SHOW ENGINE INNODB STATUS`](https://dev.mysql.com/doc/refman/5.7/en/show-engine.html "13.7.5.15 SHOW ENGINE Syntax") and [InnoDB monitor](https://dev.mysql.com/doc/refman/5.7/en/innodb-standard-monitor.html "14.17.3 InnoDB Standard Monitor and Lock Monitor Output")
output:

```
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

## Gap Locks

?? Gap Locks中存在一种插入意向锁(Insert Intention Lock)，在INSERT操作时产生。??

> - 通过`BETWEEN 10 and 20 FOR UPDATE` 锁定一个取件，防止其他线程在期间insert数据 
- `gap lock` 针对 
	1. 记录上没有索引
	2. 记录上有非唯一索引
- 如果记录上有`唯一索引（unique index）`，会使用 `index-record lock`，而非 `gap lock`；（查询条件没有覆盖唯一索引所有字段**除外**）
- 如何禁用`gap lock`
	1. 设置事务隔离级别为RC，transaction isolation level to READ COMMITTED
	2. 设置`innodb_locks_unsafe_for_binlog`参数为生效；enable the `innodb_locks_unsafe_for_binlog` system variable (which is now **deprecated**)

### ref

A gap lock is a lock on a gap between index records, or a lock on the gap before the first or after the last index record. For example, `SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;` prevents other transactions from inserting a value of `15` into column `t.c1`, whether or not there was already any such value in the column, because the gaps between all existing values in the range are locked.

A gap might span a single index value, multiple index values, or even be empty.

Gap locks are part of the tradeoff between performance and concurrency, and are used in some transaction isolation levels and not others.

Gap locking is not needed for statements that lock rows using a unique index to search for a unique row. (This does not include the case that the search condition includes only some columns of a multiple-column unique index; in that case, gap locking does occur.) For example, if the `id` column has a unique index, the following statement uses only an index-record lock for the row having `id` value 100 and it does not matter whether other sessions insert rows in the preceding gap:

```
SELECT * FROM child WHERE id = 100;
```

If `id` is not indexed or has a nonunique index, the statement does lock the preceding gap.

It is also worth noting here that conflicting locks can be held on a gap by different transactions. For example, transaction A can hold a shared gap lock (gap S-lock) on a gap while transaction B holds an exclusive gap lock (gap X-lock) on the same gap. The reason conflicting gap locks are allowed is that if a record is purged from an index, the gap locks held on the record by different transactions must be merged.

Gap locks in `InnoDB` are “purely inhibitive”, which means they only stop other transactions from inserting to the gap. They do not prevent different transactions from taking gap locks on the same gap. Thus, a gap X-lock has the same effect as a gap S-lock.

Gap locking can be disabled explicitly. This occurs if you change the transaction isolation level to [`READ COMMITTED`](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html#isolevel_read-committed) or enable the [`innodb_locks_unsafe_for_binlog`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_locks_unsafe_for_binlog) system variable (which is now deprecated). Under these circumstances, gap locking is disabled for searches and index scans and is used only for foreign-key constraint checking and duplicate-key checking.

There are also other effects of using the [`READ COMMITTED`](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html#isolevel_read-committed) isolation level or enabling [`innodb_locks_unsafe_for_binlog`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_locks_unsafe_for_binlog).
Record locks for nonmatching rows are released after MySQL has evaluated the `WHERE` condition. For
`UPDATE` statements, `InnoDB` does a “semi-consistent” read, such that it returns the latest committed version to MySQL so that MySQL can determine whether the row matches the `WHERE` condition of the [`UPDATE`](https://dev.mysql.com/doc/refman/5.7/en/update.html "13.2.11 UPDATE Syntax").

## Next-Key Locks

> Next-Key Lock = Record Lock + Gap Lock

>

### ref

A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.

`InnoDB` performs row-level locking in such a way that when it searches or scans a table index, it sets shared or exclusive locks on the index records it encounters. Thus, the row-level locks are actually index-record locks. A next-key lock on an index record also affects the “gap” before that index record. That is, a next-key lock is an index-record lock plus a gap lock on the gap preceding the index record. If one session has a shared or exclusive lock on record `R` in an index, another session cannot insert a new index record in the gap immediately before `R` in the index order.

Suppose that an index contains the values 10, 11, 13, and 20. The possible next-key locks for this index cover the following intervals, where a round bracket denotes exclusion of the interval endpoint and a square bracket denotes inclusion of the endpoint:

```
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```

For the last interval, the next-key lock locks the gap above the largest value in the index and the “supremum” pseudo-record having a value higher than any value actually in the index. The supremum is not a real index record, so, in effect, this next-key lock locks only the gap following the largest index value.

By default, `InnoDB` operates in [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) transaction isolation level. In this case, `InnoDB` uses next-key locks for searches and index scans, which prevents phantom rows (see [Section 14.5.4, “Phantom Rows”](https://dev.mysql.com/doc/refman/5.7/en/innodb-next-key-locking.html "14.5.4 Phantom Rows")).

Transaction data for a next-key lock appears similar to the following in [`SHOW ENGINE INNODB STATUS`](https://dev.mysql.com/doc/refman/5.7/en/show-engine.html "13.7.5.15 SHOW ENGINE Syntax") and [InnoDB monitor](https://dev.mysql.com/doc/refman/5.7/en/innodb-standard-monitor.html "14.17.3 InnoDB Standard Monitor and Lock Monitor Output")
output:

```
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

## Insert Intetion Locks

··· TODO

## AUTO-INC Locks

> - table-level lock
- 

### ref

An `AUTO-INC` lock is a special table-level lock taken by transactions inserting into tables with
`AUTO_INCREMENT` columns. In the simplest case, if one transaction is inserting values into the table, any other transactions must wait to do their own inserts into that table, so that rows inserted by the first transaction receive consecutive primary key values.

The [`innodb_autoinc_lock_mode`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_autoinc_lock_mode) configuration option controls the algorithm used for auto-increment locking. It allows you to choose how to trade off between predictable sequences of auto-increment values and maximum concurrency for insert operations.

For more information, see [Section 14.8.1.5, “AUTO_INCREMENT Handling in InnoDB”](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html "14.8.1.5 AUTO_INCREMENT Handling in InnoDB").

## Predicate Locks for Spatial Indexes

··· TODO

# InnoDB事务模型（Transaction Model）

## 事务的基本要素（ACID）

### 原子性（Atomicity）
事务开始后所有操作，要么全部做完，要么全部不做，不可能停滞在中间环节。事务执行过程中出错，会回滚到事务开始前的状态，所有的操作就像没有发生一样。也就是说事务是一个不可分割的整体，就像化学中学过的原子，是物质构成的基本单位。
### 一致性（Consistency）
事务开始前和结束后，数据库的完整性约束没有被破坏 。比如A向B转账，不可能A扣了钱，B却没收到。
### 隔离性（Isolation）
同一时间，只允许一个事务请求同一数据，不同的事务之间彼此没有任何干扰。比如A正在从一张银行卡中取钱，在A取钱的过程结束前，B不能向这张卡转账。
### 持久性（Durability）
事务完成后，事务对数据库的所有更新将被保存到数据库，不能回滚。

## 事务的并发问题
### 脏读
事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据
### 不可重复读
> MTDB默认使用RC，而非RR

事务 A 多次读取同一数据，事务 B 在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果 不一致。

### 幻读
系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读。

> 不可重复读的和幻读很容易混淆，不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需锁住满足条件的行，解决幻读需要锁表

## 事务隔离级别（Transaction Isolation）

事务隔离级别                 |简写| 脏读    | 不可重复读 | 幻读   
---------------------- | --- | ----- | ----- | -----
读未提交（read-uncommitted） || 是     | 是     | 是    
不可重复读（read-committed）**MT default**  |RC| 否     | 是     | 是    
可重复读（repeatable-read） **默认实现** |RR| 否     | 否     | 是    
串行化（serializable）      || 否     | 否     | 否    

Locking Read
- select with for update
- update
- delete

### 事务隔离级别 - 不可重复读（read-committed）

1. InnoDB locks only index records, not the gaps before them, and thus permits the free insertion of new records next to locked records. Gap locking is only used for foreign-key constraint checking and duplicate-key checking. 
2. must use row-based binary logging.



### 可重复读（repeatable-read） 
