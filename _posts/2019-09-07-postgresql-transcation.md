---
title: PostgreSQL 之 事务
date: 2019-09-07 03:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中事务相关内容
---

SQL标准定义了四种隔离级别。

![](/images/201909/17.png)

* `脏读` 一个事务读取了另一个并行未提交事务写入的数据。
* `不可重复读` 一个事务重新读取之前读取过的数据，发现该数据已经被另一个事务（在初始读之后提交）修改。
* `幻读` 一个事务重新执行一个返回符合一个搜索条件的行集合的查询， 发现满足条件的行集合因为另一个最近提交的事务而发生了改变。
* `序列化异常` 成功提交一组事务的结果与这些事务所有可能的串行执行结果都不一致。

在PostgreSQL中，你可以请求四种标准事务隔离级别中的任意一种，但是内部只实现了三种不同的隔离级别，即 PostgreSQL 的读未提交模式的行为和读已提交相同。

# 读已提交隔离级别

同一个事务里两个相邻的SELECT命令可能看到不同的数据。每个非事务命令可以获得一个版本号。可见性基于这个版本号。

UPDATE、DELETE、SELECT FOR UPDATE和SELECT FOR SHARE命令在两个事务并发时。正在进行的更新将等待第一个更新事务提交或者回滚。

* 如果第一个更新事务回滚，那么它的作用将被忽略并且第二个事务可以继续更新最初发现的行。
* 如果第一个更新事务提交，若该行被第一个更新者删除，则第二个更新事务将忽略该行，否则第二个更新者将试图在该行的已被更新的版本上应用它的操作。该命令的搜索条件（WHERE子句）将被重新计算来看该行被更新的版本是否仍然符合搜索条件。（这边不是计算全局新的符合条件的行）。如果符合，则第二个更新者使用该行的已更新版本继续其操作。在SELECT FOR UPDATE和SELECT FOR SHARE的情况下，这意味着把该行的已更新版本锁住并返回给客户端。

# 可重复读隔离级别

一个可重复读事务中的查询可以看见在事务中第一个非事务控制语句开始时的一个快照，而不是事务中当前语句开始时的快照。

UPDATE、DELETE、SELECT FOR UPDATE和SELECT FOR SHARE命令在搜索目标行时的行为和SELECT一样： 它们将只找到在事务开始时已经被提交的行。（第一个非事务命令决定的快照）。

不过，在被找到时，这样的目标行可能已经被其它并发事务更新（或删除或锁住）。在这种情况下， 可重复读事务将等待第一个更新事务提交或者回滚（如果它还在进行中）。

* 如果第一个更新事务回滚，那么它的作用将被忽略并且可重复读事务可以继续更新最初发现的行。
* 但是如果第一个更新事务提交（并且实际更新或删除该行，而不是只锁住它），则可重复读事务将回滚并带有`ERROR:  could not serialize access due to concurrent update`消息。因为一个可重复读事务无法修改或者锁住被其他在可重复读事务开始之后的事务改变的行。这和read committed不一样。

注意只有更新事务可能需要被重试；只读事务将永远不会有序列化冲突。

# 可序列化隔离级别

这个级别为所有已提交事务模拟序列事务执行；就好像事务被按照序列一个接着另一个被执行，而不是并行地被执行。

事实上，这个给力级别完全像可重复读一样地工作，除了它会监视一些条件，这些条件可能导致一个可序列化事务的并发集合的执行产生的行为与这些事务所有可能的序列化（一次一个）执行不一致。这种监控不会引入超出可重复读之外的阻塞，但是监控会产生一些负荷，并且对那些可能导致序列化异常的条件的检测将触发一次序列化失败。


# 查看和设置数据库的事务隔离级别

* 查看全局默认事务隔离级别

```sql
SELECT setting FROM pg_settings WHERE name = 'default_transaction_isolation';
show default_transaction_isolation;
```
* 查看当前（会话或事务）的事务隔离级别

```sql
SELECT setting FROM pg_settings WHERE name = 'transaction_isolation';
SELECT current_setting('transaction_isolation');
show transaction_isolation;
```

* 设置全局默认事务隔离级别

```
修改postgresql.conf中default_transaction_isolation的配置。或者
通过 ALTER SYSTEM SET default_transaction_isolation TO 'repeatable read'; 影响postgresql.auto.conf。
```

* 设置当前会话的事务隔离级别

```sql
SET default_transaction_isolation = 'repeatable read'; --通过default_transaction_isolation影响transaction_isolation
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ; --实际上只是用SET设置default_transaction_isolation的等效体
```

* 设置当前事务的事务隔离级别

```sql
START TRANSACTION ISOLATION LEVEL REPEATABLE READ; 或者 BEGIN ISOLATION LEVEL REPEATABLE READ READ WRITE;
或者开始事务后，在事务内使用set transaction_isolation = 'repeatable read';修改。
```

注意点：

直接通过set transaction_isolation来修改会话的事务隔离级别是无法成功的：

```sql
SET transaction_isolation = 'REPEATABLE READ';  --大写无法通过语法检查
SET transaction_isolation = 'repeatable read';  --小写通过语法检查，但是不生效
```

需要修改default_transaction_isolation来影响transaction_isolation。

实际上set transaction_isolation只在事务内部起作用，反而，在事务中修改default_transaction_isolation并不会影响到transaction_isolation。

```sql
pgdba->pgdba@[local]=# show transaction_isolation;
 transaction_isolation
-----------------------
 read committed
(1 row)
 
pgdba->pgdba@[local]=# START TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION
pgdba->pgdba@[local]=# show transaction_isolation;
 transaction_isolation
-----------------------
 repeatable read
(1 row)
 
pgdba->pgdba@[local]=# set transaction_isolation = 'read uncommitted';
SET
pgdba->pgdba@[local]=# show transaction_isolation;
 transaction_isolation
-----------------------
 read uncommitted
(1 row)
 
pgdba->pgdba@[local]=# set default_transaction_isolation = 'serializable';
SET
pgdba->pgdba@[local]=# show transaction_isolation;
 transaction_isolation
-----------------------
 read uncommitted
(1 row)
 
pgdba->pgdba@[local]=# COMMIT;
COMMIT
pgdba->pgdba@[local]=# show transaction_isolation;
 transaction_isolation
-----------------------
 serializable
(1 row)
```

可以看出来：default_transaction_isolation在commit之后才会影响到transaction_isolation。





