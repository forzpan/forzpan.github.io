---
title: PostgreSQL 之 显示锁定
date: 2019-09-07 04:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中锁相关内容
---

# 表级锁

两个事务在同一时刻不能在同一个表上持有属于相互冲突模式的锁。但是，一个事务决不会和自身冲突。例如，它可以在同一个表上获得ACCESS EXCLUSIVE锁然后接着获取ACCESS SHARE锁。

非冲突锁模式可以由许多事务同时持有。

* `ACCESS SHARE` SELECT命令在被引用的表上获得一个这种模式的锁。通常，任何只读取表而不修改它的查询都将获得这种锁模式。最弱的锁，加这个锁就是为了标记想要读数据，防止
* `ROW SHARE` SELECT FOR UPDATE和SELECT FOR SHARE命令在目标表上取得一个这种模式的锁 （加上在被引用但没有SELECT FOR UPDATE/FOR SHARE的任何其他表上的ACCESS SHARE锁）。
* `ROW EXCLUSIVE` 命令UPDATE、DELETE和INSERT在目标表上取得这种锁模式（加上在任何其他被引用表上的ACCESS SHARE锁）。通常，这种锁模式将被任何修改表中数据的命令取得。
* `SHARE UPDATE EXCLUSIVE` 这种模式保护一个表不受并发模式改变和VACUUM运行的影响。由VACUUM（不带FULL）、ANALYZE、CREATE INDEX CONCURRENTLY、CREATE STATISTICS和ALTER TABLE VALIDATE以及其他ALTER TABLE的变体获得。
* `SHARE` 这种模式保护一个表不受并发数据改变的影响。由CREATE INDEX（不带CONCURRENTLY）取得。
* `SHARE ROW EXCLUSIVE` 这种模式保护一个表不受并发数据修改所影响，并且是自排他的，这样在一个时刻只能有一个会话持有它。由CREATE COLLATION、CREATE TRIGGER和很多 ALTER TABLE的很多形式所获得。
* `EXCLUSIVE` 这种模式只允许并发的ACCESS SHARE锁，即只有来自于表的读操作可以与一个持有该锁模式的事务并行处理。由REFRESH MATERIALIZED VIEW CONCURRENTLY获得。
* `ACCESS EXCLUSIVE` 这种模式保证持有者是访问该表的唯一事务。由ALTER TABLE、DROP TABLE、TRUNCATE、REINDEX、CLUSTER、VACUUM FULL和REFRESH MATERIALIZED VIEW（不带CONCURRENTLY）命令获取。ALTER TABLE的很多形式也在这个层面上获得锁。这也是未显式指定模式的LOCK TABLE命令的默认锁模式。最强锁，想做一些操作，不允许一切其他读写操作。

**综上：只有一个ACCESS EXCLUSIVE锁阻塞一个SELECT（不带FOR UPDATE/SHARE）语句。 一旦被获取，一个锁通常将被持有直到事务结束。但是如果在建立保存点之后才获得锁，那么在回滚到这个保存点的时候将立即释放该锁。这与ROLLBACK取消保存点之后所有的影响的原则保持一致。 同样的原则也适用于在PL/pgSQL异常块中获得的锁：一个跳出块的错误将释放在块中获得的锁。**

![](/images/201909/18.png)

# 行级锁

一个事务可能会在相同的行上保持冲突的锁，甚至是在不同的子事务中。但是除此之外，两个事务永远不可能在相同的行上持有冲突的锁。行级锁不影响数据查询，它们只阻塞对同一行的写入者和加锁者。

* FOR UPDATE
* FOR NO KEY UPDATE
* FOR SHARE
* FOR KEY SHARE

![](/images/201909/19.png)

# 页级锁

页面级别的共享/排他锁被用来控制对共享缓冲池中表页面的读/写。

# 例子

```sql
BEGIN;
SELECT * FROM mytable WHERE key = 1 FOR UPDATE;
UPDATE mytable SET ... WHERE key = 1;            -- read committed这句仍然可能阻塞，因为上一句for update 锁不住并发事务新增的符合条件的行。
ROLLBACK;
```

