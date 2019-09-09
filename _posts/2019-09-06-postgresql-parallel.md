---
title: PostgreSQL 之 并行执行
date: 2019-09-06 05:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中并行执行
---

```sql
EXPLAIN SELECT * FROM pgbench_accounts WHERE filler LIKE '%x%';
                                     QUERY PLAN                                     
-------------------------------------------------------------------------------------
 Gather  (cost=1000.00..217018.43 rows=1 width=97)
   Workers Planned: 2
   ->  Parallel Seq Scan on pgbench_accounts  (cost=0.00..216018.33 rows=1 width=97)
         Filter: (filler ~~ '%x%'::text)
(4 rows)
```

当查询执行期间到达Gather节点时， 实现用户会话的进程将会请求和规划器选中的工作者数量一样多的 [后台工作者进程](http://www.postgres.cn/docs/11/bgworker.html) 。 规划器考虑使用的后台工作者的数量限制为最多 [max_parallel_workers_per_gather](http://www.postgres.cn/docs/11/runtime-config-resource.html#GUC-MAX-PARALLEL-WORKERS-PER-GATHER) 。默认为2。 

任何时候能够存在的后台工作者进程的总数由 [max_worker_processes](http://www.postgres.cn/docs/11/runtime-config-resource.html#GUC-MAX-WORKER-PROCESSES) 和 [max_parallel_workers](http://www.postgres.cn/docs/11/runtime-config-resource.html#GUC-MAX-PARALLEL-WORKERS) 限制， 因此一个并行查询可能会使用比规划中少的工作者来运行， 甚至有可能根本不使用工作者。

为一个给定并行查询成功启动的后台工作者进程都将会执行计划的并行部分。 这些工作者的领导者也将执行该计划，不过它还有一个额外的任务： 它还必须读取所有由工作者产生的元组。

当整个计划的并行部分只产生了少量元组时， 领导者通常将表现为一个额外的加速查询执行的工作者。

反过来， 当计划的并行部分产生大量的元组时，领导者将几乎全用来读取由工作者产生的元组并且执行 Gather节点或Gather Merge 节点上层计划节点所要求的任何进一步处理。

在这些情况下， 领导者所作的执行并行部分的工作将会很少。

当计划平行部分顶部的节点是Gather Merge而不是Gather时， 它表示执行计划的并行部分的每个进程正在按排序顺序生成元组， 领导者正在执行顺序保留合并。相反，Gather 以任何方便地顺序从工作者读取元组，从而破坏可能存在的任何排序顺序。

# 运行并行查询的配置

* [max_parallel_workers_per_gather](http://www.postgres.cn/docs/11/runtime-config-resource.html#GUC-MAX-PARALLEL-WORKERS-PER-GATHER) 必须被设置为大于零的值。

* [dynamic_shared_memory_type](http://www.postgres.cn/docs/10/runtime-config-resource.html#GUC-DYNAMIC-SHARED-MEMORY-TYPE) 必须被设置为除none之外的值。

* 系统一定不能运行在单用户模式下。因为在单用户模式下，整个数据库系统运行在单个进程中，没有后台工作者进程可用。

**如果下面的任一条件为真，即便对一个给定查询通常可以产生并行查询计划，规划器都不会为它产生并行查询计划：**

* 查询要写任何数据或者锁定任何数据库行。如果一个查询在顶层或者 CTE（通用表达式） 中包含了数据修改操作，那么不会为该查询产生并行计划。

* 查询可能在执行过程中被暂停。只要在系统认为可能发生部分或者增量式执行，就不会产生并行计划。例如：用DECLARE CURSOR创建的游标将永远不会使用并行计划。类似地，一个FOR x IN query LOOP .. END LOOP形式的 PL/pgSQL 循环也永远 不会使用并行计划，因为当并行查询进行时，并行查询系统无法验证循环中的代码执行起来是安全的。

* 使用了任何被标记为PARALLEL UNSAFE的函数的查询。大多数系统定义的函数都被标记为PARALLEL SAFE，但是用户定义的函数默认被标记为PARALLEL UNSAFE。

* 该查询运行在另一个已经存在的并行查询内部。

* 事务隔离级别是可串行化。

**即使对于一个特定的查询已经产生了并行查询计划，在一些情况下执行时也不会并行执行该计划。如果发生这种情况，那么领导者将会自己执行该计划在Gather节点之下的部分，就好像Gather节点不存在一样。**

上述情况将在满足下面的任一条件时发生：

* 因为后台工作者进程的总数不能超过max_worker_processes，导致不能得到后台工作者进程。

* 由于为并行查询而启动的后台工作者总数不能超过 max_parallel_workers，因此无法获得后台工作者。

* 客户端发送了一个执行消息，并且消息中要求取元组的数量不为零。执行消息可见扩展查询协议中的讨论。因为libpq当前没有提供方法来发送这种消息，所以这种情况只可能发生在不依赖 libpq 的客户端中。如果这种情况经常发生，那在它可能发生的会话中将max_parallel_workers_per_gather设置为0是一个很好的主意，这样可以避免产生连续运行时次优的查询计划。

* 准备好的语句是使用CREATE TABLE .. AS EXECUTE ..语句执行的。 此构造将本来只是只读操作的内容转换为读写操作，使其不适合并行查询。

* 事务隔离级别是可串行化。这种情况通常不会出现，因为当事务隔离级别是可串行化时不会产生并行查询计划。不过，如果在产生计划之后并且在执行计划之前把事务隔离级别改成可串行化，这种情况就有可能发生。

# 并行扫描

* 在并行顺序扫描中，表格的块将在协作过程中分配。 块一次发放一个，所以访问表仍然是连续的。

* 在一个并行位图堆扫描中，选择一个进程作为领导者。 该进程执行一个或多个索引的扫描并构建指示需要访问哪些表块的位图。 然后这些块在并行顺序扫描中在协作进程中分配。换句话说， 堆扫描是并行执行的，但底层索引扫描不是。

* 在并行索引扫描或并行仅索引的扫描中， 协作进程轮流从索引读取数据。目前， 并行索引扫描仅支持btree索引。每个进程将声明一个索引块， 并将扫描并返回该块引用的所有元组；其他进程可以同时从不同的索引块返回元组。 并行btree扫描的结果以每个工作进程内的排序顺序返回。

# 并行连接
# 并行聚合

如果我们想要一个查询能产生并行计划但事实上又没有产生，可以尝试减小 parallel_setup_cost 或者 parallel_tuple_cost 。

规划器把查询中涉及的操作分类成`并行安全`、`并行受限`或者`并行不安全`。

`并行安全`的操作不会与并行查询的使用产生冲突。

`并行受限`的操作不能在并行工作者中执行，但是能够在并行查询的领导者中执行。 因此，并行受限的操作不能出现在Gather或Gather Merge节点之下， 但是能够出现在包含有这样节点的计划的其他位置。

`并行不安全`的操作不能在并行查询中执行，甚至不能在领导者中执行。当一个查询包含任何并行不安全操作时，并行查询对这个查询是完全被禁用的。

下面的操作总是并行受限的。

* 公共表表达式（CTE）的扫描。

* 临时表的扫描。

* 外部表的扫描，除非外部数据包装器有一个IsForeignScanParallelSafe API。

* 对InitPlan或者相关的SubPlan的访问。












