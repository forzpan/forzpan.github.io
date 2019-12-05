---
title: PostgreSQL 之 分区表
date: 2019-12-05 01:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: 介绍PostgreSQL中分区表
---

基于postgre 11文档。

分区指的是将逻辑上的一个大表分成一些小的物理上的片。分区有很多益处：
* 在某些情况下查询性能能够显著提升，特别是当那些访问压力大的行在一个分区或者少数几个分区时。划分可以取代索引的主导列、减小索引尺寸以及使索引中访问压力大的部分更有可能被放在内存中。
* 当查询或更新访问一个分区的大部分行时，可以通过该分区上的一个顺序扫描来取代分散到整个表上的索引和随机访问，这样可以改善性能。
* 如果需求计划使用划分设计，可以通过增加或移除分区来完成批量载入和删除。执行`ALTER TABLE DETACH PARTITION`或使用`DROP TABLE`删除一个单独的分区都远快于一个批量操作。这些命令也完全避免了由批量`DELETE`造成的`VACUUM`负载。
* 很少使用的数据可以被迁移到便宜且较慢的存储介质上。

分区表的分区也可以是分区表，实现多级分区。

常规表不能和分区表转换，但是包含数据的常规表和分区表可以添加为分区表的分区，或者从分区表的分区剔除成为原来的形式。`ATTACH PARTITION`和`DETACH PARTITION`子命令。

分区表底层实质也是继承，对继承做了封装。分区不能拥有除分区表之外的其他父项，普通表也不能从分区表继承。

最重要的是：
* 分区表的`CHECK`和`NOT NULL`约束总是被它的所有分区继承。不允许在分区表上创建标记为`NO INHERIT`的`CHECK`约束。
* 如果`NOT NULL`约束存在于父表上，那么你不能删除分区字段上的该约束。
* 当没有分区时，仅支持使用`ONLY`在分区表上添加或删除约束。 一旦存在分区，使用`ONLY`会导致错误， 因为仅支持在分区表上添加或删除约束，当分区存在时是不支持的。相反，当不存在父表时，可以直接在分区上添加或删除约束。由于分区表不直接拥有任何数据，因此尝试在分区表上使用`TRUNCATE ONLY`将始终返回错误。
* 分区不能拥有父表中不存在的列。在使用`CREATE TABLE`创建分区时不能声明字段，也不能在创建完分区后使用`ALTER TABLE`添加字段。仅当表的列准确匹配分区表，包括oid列时，才可以使用`ALTER TABLE ... ATTACH PARTITION`将该表添加为分区。
* 分区也可以是外表（参阅[CREATE FOREIGN TABLE](http://www.postgres.cn/docs/10/sql-createforeigntable.html)）， 尽管它们会有一些普通表没有的限制。

目前支持`range`，`list`，`hash`三种分区方式

更新行的分区键可能会导致将其移动到此行满足分区范围的其他分区中。（postgre11：自动移动行到正确分区）

postgre11：与分区键不匹配的数据引入了一个默认分区。待测试。

# 内置分区表:

* 范围分区

需要为分区创建描述分区范围条件的表约束。 在需要引用时，分区约束是隐式的从分区范围声明中生成的。

```sql
CREATE TABLE measurement (
    city_id         int not null,
    logdate         date not null,
    peaktemp        int,
    unitsales       int
) PARTITION BY RANGE (logdate);
 
CREATE TABLE measurement_y2006m02 PARTITION OF measurement
    FOR VALUES FROM ('2006-02-01') TO ('2006-03-01')
 
CREATE TABLE measurement_y2006m03 PARTITION OF measurement
    FOR VALUES FROM ('2006-03-01') TO ('2006-04-01')
 
...
CREATE TABLE measurement_y2007m11 PARTITION OF measurement
    FOR VALUES FROM ('2007-11-01') TO ('2007-12-01')
 
CREATE TABLE measurement_y2007m12 PARTITION OF measurement
    FOR VALUES FROM ('2007-12-01') TO ('2008-01-01')
    TABLESPACE fasttablespace;
 
CREATE TABLE measurement_y2008m01 PARTITION OF measurement
    FOR VALUES FROM ('2008-01-01') TO ('2008-02-01')
    TABLESPACE fasttablespace
    WITH (parallel_workers = 4);
```

要实现子分区，在创建单个分区的语句中声明`PARTITION BY`子句，例如：

```sql
CREATE TABLE measurement_y2006m02 PARTITION OF measurement
    FOR VALUES FROM ('2006-02-01') TO ('2006-03-01')
    PARTITION BY RANGE (peaktemp);
```

对于每一个分区，在键列上创建索引，以及您可能需要的其他索引。

```sql
CREATE INDEX ON measurement_y2006m02 (logdate);
CREATE INDEX ON measurement_y2006m03 (logdate);
...
CREATE INDEX ON measurement_y2007m11 (logdate);
CREATE INDEX ON measurement_y2007m12 (logdate);
CREATE INDEX ON measurement_y2008m01 (logdate);
```

确保在`postgresql.conf`中`enable_partition_pruning`配置参数`没有被禁用`。如果它被禁用，查询将不会被按照期望的方式优化。

移除旧数据的最简单的选项是删除不再需要的分区：

```sql
DROP TABLE measurement_y2006m02;
```

不过，请注意，上述命令需要在父表上获取一个`ACCESS EXCLUSIVE`锁。

另一个经常使用的选项是将分区从被划分的表中移除，但是把它作为一个独立的表保留下来：

```sql
ALTER TABLE measurement DETACH PARTITION measurement_y2006m02;
```

相似地我们也可以增加新分区来处理新数据。我们可以在被划分的表中创建一个新的空分区：

```sql
CREATE TABLE measurement_y2008m02 PARTITION OF measurement
    FOR VALUES FROM ('2008-02-01') TO ('2008-03-01')
    TABLESPACE fasttablespace;
```

作为一种选择方案，有时创建一个在分区结构之外的新表更方便，并且在以后才将它作为一个合适的分区。这使得数据可以在出现于被划分表中之前被载入、检查和转换：

```sql
CREATE TABLE measurement_y2008m02
  (LIKE measurement INCLUDING DEFAULTS INCLUDING CONSTRAINTS)
  TABLESPACE fasttablespace;
 
ALTER TABLE measurement_y2008m02 ADD CONSTRAINT y2008m02
   CHECK ( logdate >= DATE '2008-02-01' AND logdate < DATE '2008-03-01' );
 
\copy measurement_y2008m02 from 'measurement_y2008m02'
-- 可能做一些其他数据准备工作
 
ALTER TABLE measurement ATTACH PARTITION measurement_y2008m02
    FOR VALUES FROM ('2008-02-01') TO ('2008-03-01' );
```

在运行`ATTACH PARTITION`命令之前，建议在要附加的表上创建一个`CHECK约束`来描述所需的分区约束。这样，系统将能够跳过扫描来验证隐式分区约束。如果没有这样的约束，将在父表上保存一个`ACCESS EXCLUSIVE`锁来扫描该表以验证分区约束。然后可以在`ATTACH PARTITION`完成后删除约束，因为它不再是必需的。

以下限制适用于分区表：
* 无法创建跨越所有分区的排除约束; 它只能单独约束每个叶子分区。（postgre11：能够在传递给所有分区的分区表上创建主键、外键、索引和触发器）
* 虽然分区表支持主键，但不支持引用分区表的外键。（支持从分区表到其他表的外键引用。）
* 对分区表使用`ON CONFLICT`子句会导致错误， 因为唯一或排除约束只能在单个分区上创建。不支持在整个分区层次结构中实施唯一性（或排除约束）。（postgre11：这句删除）
* 当一个`UPDATE`操作导致行从一个分区移动到另一个分区时，另一个并发`UPDATE`或`DELETE`可能错过该行。假设会话1正在对分区key执行`update`，同时该行可见的并发会话2对该行执行`UPDATE`或`DELETE`操作。如果由于会话1的活动而从分区中删除了行，则会话2可以默默地丢失该行。在这种情况下，会话2`UPDATE`或`DELETE`，不知道行移动认为该行刚被删除，并得出结论，该行没有任何事情要做。在通常不对表进行分区的情况下，或者没有行移动的情况下，会话2将识别出新更新的行并在该新行版本上执行`UPDATE`/`DELETE`。
* `BEFORE ROW`必要时，必须在各个分区上定义触发器，而不是在分区表上定义。（postgre11：不是能够传递吗？）
* 不允许在同一分区树中混合临时和永久关系。因此，如果分区表是永久性的，则分区也必须是永久的，并且如果分区表是临时的，则分区也必须是临时的。使用临时关系时，分区树的所有成员必须来自同一会话。

# 使用继承实现分区表

不推荐用了。

PG12开始，分区表性能大幅提升，分区插件可以扔了吗？


