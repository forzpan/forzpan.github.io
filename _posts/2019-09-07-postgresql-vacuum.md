---
title: PostgreSQL 之 VACUUM
date: 2019-09-07 01:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中VACUUM相关内容
---

PG使用MVCC，废弃的行版本对应着很多的dead tuple，表的堆文件会膨胀，主要是为了解决这个问题，引入VACUUM清理机制。有 标准VACUUM 和 VACUUM FULL 两种形式。

# VACUUM的主要功能：

## 恢复磁盘空间

对于将被已更新或已删除行所占用的磁盘空间，需要回收。

标准VACUUM，移除表和索引中的死亡行版本并将该空间标记为可在未来重用，不会把空间交还操作系统，除非表尾部的连续页面完全空闲并且能很容易地得到一个排他表锁。

VACUUM FULL通过把死亡空间之外的内容写成一个完整的新版本表文件来主动紧缩表。需要额外的空间。表上需要排他锁。VACUUM FULL，或者CLUSTER，或者ALTER TABLE的表重写都是重新创建底层物理文件。

标准VACUUM不影响CRUD，因此例行VACUUM尽量走标准VACUUM。实际autovacuum不会用VACUUM FULL。

## 更新规划器统计信息

一般有analyze命令来收集信息，作为VACUUM的一个可选功能，可以及时的更新信息，帮助选择正确的执行计划。

对于重度更新的表，才需要频繁的调用analyze，这里的重度更新是指条件字段值区间的迅速扩张，对于稳定分布的数据（平均分布，正态分布等），频繁的收集信息，效果不大。

通过ALTER TABLE SET STATISTICS，或者使用default_statistics_target配置参数改变数据库范围的默认值,可以获取更细粒度的信息。

## 更新可见性映射

每一个堆关系都有一个可见性映射，文件名用_vm结尾。可见性映射仅为每个堆页面存储两个位。

第一位如果被设置，表示该页面上的元组都是可见的，或者换句话说该页面不含有任何需要被清理的元组。这些信息也可以被用于index-only scans。

第二位如果被设置，表示该页面上的元组都已经被冻结。这也意味着防回卷清理操作也不需要重新访问该页面。

该映射是保守的，我们可以确定不论何时一个位被设置，那就说明条件为真，但是如果一个位没有被设置，它可能为真也可能不为真。

可见性映射的位只会被清理操作设置，但是可以被任何在页面上进行的数据修改操作清除。

delete语句会清除  都可见bit 。

insert语句会清除  都冻结bit 。

update语句由于mvcc，可以理解成delete和insert，修改在两个bit可能属于一个堆页面，也可能是两个堆页面。

vacuum时，扫描一个堆页面，依次扫描各个元组，清理所有死亡元组，然后设置第一位，如果元组xid需要冻结，加上冻结标记，如果所有元组都被冻结，设置第二位。

这个两个bit的关系：都可见不代表都冻结，没有那么老xid，就没必要冻结；都冻结不带表都可见，说不定冻结的行，后来被废弃。

## 防止事务 ID 回卷失败

对于xid，是个uint32类型，40亿多点。循环使用，将xid看成一个环，想像成一个时钟，对于一个普通xid，使用到之前，如果它已经被使用，则必须用一个特殊的标志（冻结事务ID：2）来标记，9.4之前是替换xmin，新版本只是设置一个标志位。

【0：无效事务ID；1：初始化事务ID；2：冻结事务ID；3：最小事务ID】它们被认为比任何普通xid都老。

显然用到了才冻结是比较紧急且危险的事情，对于普通xid，pg使用modulo-2^32算法，总是认为后面20亿个xid是未来，前面20亿个xid是过去，为了避免发生回卷，有必要最多每20亿个事务就清理每个数据库中的每个表。

# 相关参数 

标准VACUUM也分低代价的part scan和高代价的full scan。part scan会跳过不含有任何死亡行版本的页面（即可见性映射第一位被设置）。问题是那些含有旧xid的堆页面怎样及时冻结它呢？

答案：要保证所有旧的行版本都已经被冻结，需要对整个表做一次full scan（可见性映射第二位被设置的堆页面会被跳过），显然这是一个需要反复做的操作，需要一个触发机制。

因此有一个机制：

首先理解一下relfrozenxid：一个表的pg_class.relfrozenxid是该表上一次全表VACUUM所用的冻结截止XID。该表中所有比这个截断 XID 老的普通 XID 的事务插入的行 都确保被冻结。

* **autovacuum_freeze_max_age**

　　指定在一个VACUUM操作被强制执行来防止表中事务ID回卷之前，一个表的pg_class.relfrozenxid域能保持的最大年龄。

　　即一旦newestxid - relfrozenxid > autovacuum_freeze_max_age则系统会强制vacuum，即使autovacuum没开启。

* **vacuum_freeze_table_age**

　　当表的pg_class.relfrozenxid域达到该设置指定的年龄时（newestxid - relfrozenxid），VACUUM会执行一次全表扫描。VACUUM会悄悄地将有效值限制为autovacuum_freeze_max_age值的95%。

　　也就是一旦newestxid - relfrozenxid > vacuum_freeze_table_age（最大0.95*autovacuum_freeze_max_age）则进行一次全表扫描。

　　95%是一种经验法则，留出一个足够的空间让一次被正常调度的VACUUM或一次被正常删除和更新活动触发的自动清理可以在这个窗口中被运行。

　　将它设置得太接近autovacuum_freeze_max_age可能导致防回卷自动清理，即使该表最近因为回收空间的目的被清理过，而较低的值将导致更频繁的全表扫描。

　　将这个参数设置为 0 将强制VACUUM总是扫描所有页面而实际上忽略可见性映射。

* **vacuum_freeze_min_age**

　　指定VACUUM在扫描表时用来决定是否冻结行版本的临界年龄。VACUUM会悄悄地将有效值限制为autovacuum_freeze_max_age值的一半。

　　即一旦newestxid - xmin > vacuum_freeze_min_age 即 xmin < newestxid - vacuum_freeze_min_age 就可以冻结该行。

# 处理机制

以上几个参数结合理解一下：

　　手动或者自动触发full scan，以每次full scan时间点的newestxid为基准，从旧relfrozenxid到newestxid遇到的第一个未提交的xid为oldestxid。

　　当 oldestxid - vacuum_freeze_min_age > 旧relfrozenxid 则 full scan完成后的 新relfrozenxid =  oldestxid - vacuum_freeze_min_age 否则就是 oldestxid。（这是我猜的）。

　　再下一次自动full scan发生应该在 新relfrozenxid后的vacuum_freeze_table_age个事务。也就是 oldestxid  后的 vacuum_freeze_table_age - vacuum_freeze_min_age 个事务。

　　oldestxid 最大是 newestxid ，那么 也就是 newestxid 之后最多 vacuum_freeze_table_age - vacuum_freeze_min_age 个事务。

　　所以说每 vacuum_freeze_table_age - vacuum_freeze_min_age 个事务，肯定会之地执行一次full scan，而不是每vacuum_freeze_table_age事务。

　　如果一个表没有被启用autovacuum，大约每autovacuum_freeze_max_age - vacuum_freeze_min_age事务就会在该表上强制调用一次清理。

增加autovacuum_freeze_max_age（以及类似作用的vacuum_freeze_table_age）的唯一不足是数据库集簇的pg_xact和pg_commit_ts子目录将占据更多空间，因为它必须存储所有relfrozenxid向后autovacuum_freeze_max_age范围内的所有事务的提交状态和（如果启用了track_commit_timestamp）时间戳。提交状态为每个事务使用两个二进制位，因此如果autovacuum_freeze_max_age被设置为它的最大允许值 20 亿，pg_xact将会增长到大约 0.5 吉字节（2 * 2 000 000 000 / 8 = 5 00 000 000），pg_commit_ts大约20GB(一个时间戳10byte？)。如果这对于你的总数据库尺寸是微小的，我们推荐设置autovacuum_freeze_max_age为它的最大允许值。否则，基于你想要允许pg_xact和pg_commit_ts使用的存储空间大小来设置它（默认情况下 2 亿个事务大约等于pg_xact的 50 MB存储空间，pg_commit_ts的2GB的存储空间）。

# autovacuum工作过程：

todo

# 参考链接

[PostgreSQL vacuum原理一功能与参数](http://blog.itpub.net/30088583/viewspace-1592066)

[PostgreSQL vacuum原理—启动机制](http://blog.itpub.net/30088583/viewspace-1601849)

[PostgreSQL vacuum原理—vacuum揭秘](http://blog.itpub.net/30088583/viewspace-1603893)

[PostgreSQL vacuum 内核源码机理](http://blog.itpub.net/30088583/viewspace-1618496)

[PostgreSQL Cost Based Vacuum探秘](http://blog.itpub.net/30088583/viewspace-1615226)

[PgSQL · 源码分析 · AutoVacuum机制之autovacuum launcher](http://mysql.taobao.org/monthly/2017/12/04)

[PgSQL · 源码分析 · AutoVacuum机制之autovacuum worker](http://mysql.taobao.org/monthly/2018/02/04)

[MVCC and VACUUM](https://yq.aliyun.com/articles/662475)

[PostgreSQL 9.6 vacuum freeze大幅性能提升 代码浅析](https://github.com/digoal/blog/blob/master/201610/20161002_03.md)








