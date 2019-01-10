---
title: mysql 死锁的那些事儿
date: 2018-03-10 16:46:55
tags:
- mysql
- deadlock
---

> 快捷入口
- 在 `show engine innodb status\G` 中查询 `LATEST DETECTED DEADLOCK`;
- 通过 `SELECT * FROM information_schema.INNODB_LOCKS;` 查询锁信息；
- `begin`, `commit` 的方式手动触发，复现问题

# 查看当前事务隔离级别

```
mysql> SELECT @@global.tx_isolation, @@session.tx_isolation, @@tx_isolation;
+-----------------------+------------------------+----------------+
| @@global.tx_isolation | @@session.tx_isolation | @@tx_isolation |
+-----------------------+------------------------+----------------+
| READ-COMMITTED        | READ-COMMITTED         | READ-COMMITTED |
+-----------------------+------------------------+----------------+
1 row in set (0.00 sec)
```

# 查看当前数据库锁

## 通过进程信息查询

```
show full processlist
```

主要观察state和info

## 查询 innodb status

```
show engine innodb status\G
```

重点关注 `TRANSACTIONS` 部分和 `LATEST DETECTED DEADLOCK` 两个部分；

### 通过 `information_shcema` 数据表查询

- innodb_trx		：innodb内核中的当前活跃（ACTIVE）事务
- innodb_locks  	：当前状态产生的innodb锁 仅在有锁等待时
- innodb_lock_waits	：当前状态产生的innodb锁等待 仅在有锁等待时

#### innodb_trx 表结构
字段名                   | 说明                       
--------------------- | -----------------------------------
trx_id|innodb存储引擎内部唯一的事物ID
trx_state  | 当前事物状态（running和lock wait两种状态）
trx_started         |   事物的开始时间 
trx_requested_lock_id | 等待事物的锁ID，如trx_state的状态为Lock wait，那么该值带表当前事物等待之前事物占用资源的ID，若trx_state不是Lock wait 则该值为NULL
trx_wait_started      | 事物等待的开始时间
trx_weight            | 事物的权重，在innodb存储引擎中，当发生死锁需要回滚的时，innodb存储引擎会选择该值最小的进行回滚
trx_mysql_thread_id   | mysql中的线程id, 即show  processlist显示的结果
trx_query            |  事物运行的SQL语句

#### innodb_locks 表结构

字段名         | 说明  
----------- | ----------------
lock_id    |  锁的ID
lock_trx_id | 事物的ID
lock_mode   | 锁的模式（S锁与X锁两种模式）
lock_type   | 锁的类型 表锁还是行锁（RECORD）
lock_table  | 要加锁的表
lock_index  | 锁住的索引
lock_space  | 锁住对象的space id
lock_page  |  事物锁定页的数量，若是表锁则该值为NULL
lock_rec    | 事物锁定行的数量，若是表锁则该值为NULL
lock_data   | 事物锁定记录主键值，若是表锁则该值为NULL（此选项不可信）

#### innodb_lock_waits 表结构

字段名               | 说明   
----------------- | -----------      
requesting_trx_id | 申请锁资源的事物ID
requested_lock_id | 申请的锁的ID
blocking_trx_id   | 阻塞其他事物的事物ID
blocking_lock_id  | 阻塞其他锁的锁ID

### 如何读 `LATEST DETECTED DEADLOCK`

`LATEST DETECTED DEADLOCK` 中会记录最近一次死锁的信息，包含以下信息
- 包含两个事务：TRANSACTION 119992026，TRANSACTION 119991521
- 事务2026
  1. <font color=#0099ff>*等待*</font> 一个行级的 <font color=#0099ff>**X锁**</font> （`WAITING FOR ···· RECORD LOCKS ··· lock_mode X locks rec but not gap`）
- 事务1521
  1. <font color=#0099ff>*持有*</font> 一个行级的 <font color=#0099ff>**X锁**</font> （`HOLDS ··· RECORD LOCKS ··· lock_mode X locks rec but not gap`）
  2. <font color=#0099ff>*等待*</font> 一个行级的 <font color=#0099ff>**S锁**</font> （`WAITING FOR THIS LOCK TO BE GRANTED ··· RECORD LOCKS ··· lock mode S waiting`）

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2018-03-10 16:21:02 0x7f8ac30e1700
*** (1) TRANSACTION:
TRANSACTION 119992026, ACTIVE 87 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 1462608, OS thread handle 140233934087936, query id 4903877046 172.26.182.190 root updating
delete from ac_role_function WHERE (  role_id = 12287 and business_sys_id = 353 and tenant_code = '1' )
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 9190 page no 35 n bits 448 index unique_busi_tenant_role_fun of table `uac_test`.`ac_role_function` trx id 119992026 lock_mode X locks rec but not gap waiting
*** (2) TRANSACTION:
TRANSACTION 119991521, ACTIVE 149 sec inserting
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 2
MySQL thread id 1462607, OS thread handle 140233954694912, query id 4903902025 172.26.182.190 root update
insert into ac_role_function ( tenant_code, business_sys_id, role_id, function_id ) values ( '1', 353, 12287, 2869 )
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 9190 page no 35 n bits 448 index unique_busi_tenant_role_fun of table `uac_test`.`ac_role_function` trx id 119991521 lock_mode X locks rec but not gap
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 9190 page no 35 n bits 448 index unique_busi_tenant_role_fun of table `uac_test`.`ac_role_function` trx id 119991521 lock mode S waiting
*** WE ROLL BACK TRANSACTION (1)
```

# 🌰

## 案发现场

### innodb status 中 deadlock 信息
事务2026占用X锁失败，
```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2018-03-10 16:21:02 0x7f8ac30e1700
*** (1) TRANSACTION:
TRANSACTION 119992026, ACTIVE 87 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 1462608, OS thread handle 140233934087936, query id 4903877046 172.26.182.190 root updating
delete from ac_role_function WHERE (  role_id = 12287 and business_sys_id = 353 and tenant_code = '1' )
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 9190 page no 35 n bits 448 index unique_busi_tenant_role_fun of table `uac_test`.`ac_role_function` trx id 119992026 lock_mode X locks rec but not gap waiting
*** (2) TRANSACTION:
TRANSACTION 119991521, ACTIVE 149 sec inserting
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 2
MySQL thread id 1462607, OS thread handle 140233954694912, query id 4903902025 172.26.182.190 root update
insert into ac_role_function ( tenant_code, business_sys_id, role_id, function_id ) values ( '1', 353, 12287, 2869 )
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 9190 page no 35 n bits 448 index unique_busi_tenant_role_fun of table `uac_test`.`ac_role_function` trx id 119991521 lock_mode X locks rec but not gap
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 9190 page no 35 n bits 448 index unique_busi_tenant_role_fun of table `uac_test`.`ac_role_function` trx id 119991521 lock mode S waiting
*** WE ROLL BACK TRANSACTION (1)
```

### 业务系统记下了这个bug

> Deadlock found when trying to get lock; try restarting transaction

证据确凿，一个deadlock

```
### SQL: delete from ac_role_function WHERE (  role_id = ? and business_sys_id = ? and tenant_code = ? )
### Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
; SQL []; Deadlock found when trying to get lock; try restarting transaction; nested exception is com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
        at org.springframework.jdbc.support.SQLErrorCodeSQLExceptionTranslator.doTranslate(SQLErrorCodeSQLExceptionTranslator.java:263)
        at org.springframework.jdbc.support.AbstractFallbackSQLExceptionTranslator.translate(AbstractFallbackSQLExceptionTranslator.java:73)
        at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:74)
        at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:421)
        at com.sun.proxy.$Proxy52.delete(Unknown Source)
```

## 到底发生了什么

这段业务代码中在进行一个关联信息的保存操作
- 按照上级先删除所有的关联信息；
- 按照用户当前的配置，插入新的关联信息；
- 关联信息建有唯一索引

然后，坑出现了！！！

- 删除操作需要占用资源的X锁
- 执行插入操作前，需要占用资源的S锁；

当请求并发执行时
- 事务1：占用资源的X锁，删除资源；
- 事务2：占用资源的X锁，删除资源（此时删除操作被挂起，等待X锁）
- 事务1：执行插入操作，请求资源的S锁；
- 事务1执行完成，事务2直接抛出deadlock

## sql命令行验证下假想

时序|事务1（1521）|事务2（2026）|锁状态
---|---|---|---
1|begin|begin|
2|`delete from ac_role_function WHERE role_id = 12287 and business_sys_id = 353 and tenant_code = '1' ; `| |
3||```delete from ac_role_function WHERE role_id = 12287 and business_sys_id = 353 and tenant_code = '1' ;```| 事务1、事务2分别持有X锁，锁定索引相同
4|```insert into ac_role_function ( tenant_code, business_sys_id, role_id, function_id ) values ( '1', 353, 12287, 2869 );```|| 事务2抛出deadlock，事务回滚

## 然后呢

在这个案例中
- 数据是用户自身的，请求并不需要并行执行 --》 前端串行化
- X锁和S锁的矛盾源头是保存操作中，不管三七二十一的删了重新插入，导致每个请求都需要针对这些资源同时申请X锁和S锁 --》 拆分保存操作，只保存实际新增的，只删除实际需要移除的，不修改未发生变更的数据（基于log的数据同步也能降低些压力）；

# 再来个🌰，多线程插入数据引起的死锁回滚（唯一索引场景）

[引自：mysql deadlock when concurrent insert](http://www.4e00.com/blog/mysql/2017/09/16/mysql-deadlock-when-concurrent-insert.html)

## 案发现场

应用程序集群有3台机器，由于并发的问题，3台机器同时往数据里插入记录，并且在插入数据过程有其他的数据校验，因为数据校验失败，事务需要回滚，而导致死锁产生。

## 案件重演

三个事务并发执行插入请求，分别请求X锁；
事务T1发生回滚后
- 事务T3抢到X锁，执行成功；
- 事务T2没有抢到X锁，发生死锁，事务回滚

T1(140104)  | T2(140105)  | T3(140106) 
--- | --- | ---
BEGIN;   | BEGIN;| BEGIN; 
`INSERT INTO deadlock(a,b,c) VALUES(1,2,4);` ||
| `INSERT INTO deadlock(a,b,c) VALUES(1,2,4); `|
|| `INSERT INTO deadlock(a,b,c) VALUES(1,2,4);`
ROLLBACK;||
||Query OK, 1 row affected (13.10 sec)      
| ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |                                           
> 如果T1没有发生回滚，T2、T3会报唯一主键冲突；

## 看看他们是怎么处理的

- 避免此DEADLOCK
我们都知道死锁的问题通常都是业务处理的逻辑造成的，既然是uniq key，同时多台不同服务器上的相同程序对其 INSERT 一模一样的value，这本身逻辑就不太完美。
- 故解决此问题：
  1. 保证业务程序别再同一时间点并发的插入相同的值到相同的uniq key的表中
  2. 上述实验可知，是由于第一个事务ROLLBACK了才产生的DEADLOCK，查明ROLLBACK的原因
  3. 尽量减少完成事务的时间

