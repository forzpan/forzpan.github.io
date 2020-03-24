---
title: PostgreSQL 之 高性能入库
date: 2019-09-06 03:00:00
categories:
- PostgreSQL-Administrate
tags:
- PostgreSQL
description: PostgreSQL中高性能入库设置
---

# 如何设置

* **禁用自动提交**

* **使用COPY**
    * 或者使用[PREPARE](http://www.postgres.cn/docs/10/sql-prepare.html)来创建一个预备`INSERT`语句也有所帮助，然后根据需要使用`EXECUTE`多次。这样就避免了重复分析和规划`INSERT`的负荷。
    * 载入大量行时，使用`COPY`几乎总是比使用`INSERT`快， 即使用了PREPARE并且把多个插入被成批地放入一个单一事务。
    * 当[wal_level](http://www.postgres.cn/docs/11/runtime-config-wal.html#GUC-WAL-LEVEL)设置为`minimal`时，`COPY到在同一个事务中被创建或截断的表中`不需要写`WAL` 。

* **移除索引**

* **移除外键约束**

* **增加maintenance_work_mem**
    * 在载入大量数据时，临时增大[maintenance_work_mem](http://www.postgres.cn/docs/11/runtime-config-resource.html#GUC-MAINTENANCE-WORK-MEM)配置变量可以改进性能。这个参数也可以帮助加速`CREATE INDEX`命令和`ALTER TABLE ADD FOREIGN KEY`命令。 它不会对`COPY`本身起很大作用。

* **增加max_wal_size**
    * 临时增大[max_wal_size](http://www.postgres.cn/docs/11/runtime-config-wal.html#GUC-MAX-WAL-SIZE)配置变量也可以让大量数据载入更快。 这是因为向PostgreSQL中载入大量的数据将导致检查点的发生比平常（由`checkpoint_timeout`配置变量指定）更频繁。无论何时发生一个检查点时，所有脏页都必须被刷写到磁盘上。通过在批量数据载入时临时增加`max_wal_size`，所需的检查点数目可以被缩减。

* **禁用WAL归档和流复制**
    * 当使用WAL归档或流复制向一个安装中载入大量数据时，在录入结束后执行一次新的基础备份比处理大量的增量WAL数据更快。为了防止载入时记录增量WAL，通过将[wal_level](http://www.postgres.cn/docs/11/runtime-config-wal.html#GUC-WAL-LEVEL)设置为`minimal`、将[archive_mode](http://www.postgres.cn/docs/11/runtime-config-wal.html#GUC-ARCHIVE-MODE)设置为`off`以及将[max_wal_senders](http://www.postgres.cn/docs/11/runtime-config-replication.html#GUC-MAX-WAL-SENDERS)设置为`0`来禁用归档和流复制。但需要注意的是，修改这些设置需要重启服务。
    * 除了避免归档器或WAL发送者处理WAL数据的时间之外，这样做将实际上使某些命令更快， 因为它们被设计为在`wal_level`为`minimal`时完全不写WAL（通过在最后执行一个fsync而不是写WAL，它们能以更小地代价保证崩溃安全）。这适用于下列命令：　　

```sql
    CREATE TABLE AS SELECT;
    CREATE INDEX; -- 以及类似 ALTER TABLE ADD PRIMARY KEY的变体）
    ALTER TABLE SET TABLESPACE;
    CLUSTER;
    COPY FROM; -- 当目标表已经被创建或者在同一个事务的早期被截断
```

* **事后运行ANALYZE**

# 关于pg_dump的一些注记

pg_dump生成的转储脚本自动应用上面的若干个（但不是全部）技巧。 要尽可能快地载入pg_dump转储，你需要手工做一些额外的事情（请注意，这些要点适用于恢复一个转储，而不是创建它的时候。同样的要点也适用于使用psql载入一个文本转储或用pg_restore从一个pg_dump归档文件载入）。

默认情况下，pg_dump使用COPY，并且当它在生成一个完整的模式和数据转储时， 它会很小心地先装载数据，然后创建索引和外键。因此在这种情况下，一些指导方针是被自动处理的。需要做的是：

* 为`maintenance_work_mem`和`max_wal_size`设置适当的（即比正常值大的）值。
* 如果使用WAL归档或流复制，在转储时考虑禁用它们。在载入转储之前，可通过将`archive_mode`设置为`off`、将`wal_level`设置为`minimal`以及将`max_wal_senders`设置为`0`（在录入dump前）来实现禁用。 之后，将它们设回正确的值并执行一次新的基础备份。
* 采用pg_dump和pg_restore的并行转储和恢复模式进行实验并且找出要使用的最佳并发任务数量。通过使用`-j`选项的并行转储和恢复应该能为你带来比串行模式高得多的性能。
* 考虑是否应该在一个单一事务中恢复整个转储。要这样做，将`-1`或`--single-transaction`命令行选项传递给psql或pg_restore。 当使用这种模式时，即使是一个很小的错误也会回滚整个恢复，可能会丢弃已经处理了很多个小时的工作。根据数据间的相关性， 可能手动清理更好。如果你使用一个单一事务并且关闭了WAL归档，COPY命令将运行得最快。
* 如果在数据库服务器上有多个CPU可用，可以考虑使用pg_restore的`--jobs`选项。这允许并行数据载入和索引创建。
* 之后运行`ANALYZE`。

一个只涉及数据的转储仍将使用COPY，但是它不会删除或重建索引，并且它通常不会触碰外键。 因此当载入一个只有数据的转储时，如果你希望使用那些技术，你需要负责删除并重建索引和外键。

在载入数据时增加`max_wal_size`仍然有用，但是不要去增加`maintenance_work_mem`；并且不要忘记在完成后执行`ANALYZE`。

# 内存模式

* 将数据库集簇的数据目录放在一个内存支持的文件系统上（即RAM磁盘）。这消除了所有的数据库磁盘I/O，但将数据存储限制到可用的内存量（可能有交换区）。
* 关闭`fsync`；不需要将数据刷入磁盘。
* 关闭`synchronous_commit`；可能不需要在每次提交时强制把WAL写入磁盘。这种设置可能会在数据库崩溃时带来事务丢失的风险（但是没有数据破坏）。
* 关闭`full_page_writes`；不许要警惕部分页面写入。
* 增加`max_wal_size`和`checkpoint_timeout`； 这会降低检查点的频率，但会增加pg_wal的存储要求。
* 创建不做日志的表来避免WAL写入，不过这会让表在崩溃时不安全。

原链接

* [填充一个数据库](http://www.postgres.cn/docs/11/populate.html)
* [非持久设置](http://www.postgres.cn/docs/11/non-durability.html)

