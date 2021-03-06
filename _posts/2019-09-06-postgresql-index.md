---
title: PostgreSQL 之 索引
date: 2019-09-06 04:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中各种索引
---

PostgreSQL提供了多种索引类型： B-tree、Hash、GiST、SP-GiST 、GIN 和 BRIN。

# B-tree

优化器也会将B-tree索引用于涉及到模式匹配操作符LIKE和~ 的查询，前提是如果模式是一个常量且被固定在字符串的开头。使用C区域设置。

同样可以将B-tree索引用于ILIKE和~*，但仅当模式以非字母字符开始，即不受大小写转换影响的字符。

在索引列上的IS NULL或IS NOT NULL条件也可以在B-tree索引中使用。

# Hash

Hash索引只能处理简单等值比较。

# GiST

GiST索引并不是一种单独的索引，而是可以用于实现很多不同索引策略的基础设施。

和GiST相似，SP-GiST索引为支持多种搜索提供了一种基础结构。

# GIN 

GIN 索引是“倒排索引”，它适合于包含多个组成值的数据值，例如数组。

# BRIN 

BRIN 索引（块范围索引的缩写）存储有关存放在一个表的连续物理块范围上的值摘要信息。

# 须知

* 索引列上的IS NULL或IS NOT NULL条件也可以在B-tree索引中使用，优化器也会将B-tree索引用于涉及到模式匹配操作符LIKE和~ 的查询，前提是如果模式是一个常量且被固定在字符串的开头—例如：col LIKE 'foo%'或者col ~ '^foo'。但是，如果我们的数据库没有使用C区域设置，我们需要创建一个具有特殊操作符类的索引来支持模式匹配查询。同样可以将B-tree索引用于ILIKE和~*，但仅当模式以非字母字符开始，即不受大小写转换影响的字符。

* WHERE x = 42 OR x = 47 OR x = 53 OR x = 99 和 WHERE x = 5 AND y = 6 这种可以使用多个索引，组合merge出最后结果。为了组合多个索引，系统扫描每一个所需的索引并在内存中准备一个位图用于指示表中符合索引条件的行的位置。然后这些位图会被根据查询的需要“与”和“或”起来。最后，实际的表行将被访问并返回。表行将被以物理顺序访问，因为位图就是以这种顺序布局的。这意味着原始索引中的任何排序都会被丢失，并且如果存在一个ORDER BY子句就需要一个单独的排序步骤。由于这个原因以及每一个附加的索引都需要额外的时间，即使有额外的索引可用，规划器有时也会选择使用单一索引扫描。

* 部分索引，不是在列上substring这种，而是在部分行集的列上进行索引。create Index 可以加where条件。限制元数据+索引。甚至可以达到创建部分唯一索引的目的。

* 只有操作被对应索引类型的操作符族涉及才能够使用对应的索引，操作符类是操作符族的子集。（索引是基于规则）。一个索引在每一个索引列上只能支持一种排序规则。

CREATE INDEX name ON table (column opclass [sort options] [, ...]);操作符类会决定基本的排序顺序（可以通过增加排序选项COLLATE、ASC/DESC和/或 NULLS FIRST/NULLS LAST来修改）。

但是对PostgreSQL中的任何表扫描还有一个额外的要求：必须验证每一个检索到的行对该查询的 MVCC 快照是“可见的”，如第 13 章中讨论的那样。可见性信息并不存储在索引项中，只存储在堆项中。因此，乍一看似乎每一次行检索无论如何都会要求一次堆访问。如果表行最近被修改过，确实是这样。但是，对于很少更改的数据有一种方法可以解决这个问题。PostgreSQL为表堆中的每一个页面跟踪是否其中所有的行的年龄都足够大，以至于对所有当前以及未来的事务都可见。这个信息存储在该表的可见性映射的一个位中。在找到一个候选索引项后，只用索引的扫描会检查对应堆页面的可见性映射位。如果该位被设置，那么这一行就是可见的并且该数据库可以直接被返回。如果该位没有被设置，那么就必须访问堆项以确定这一行是否可见，这种情况下相对于标准索引扫描就没有性能优势。即便是在成功的情况下，这种方法也是把对堆的访问换成了对可见性映射的访问。不过由于可见性映射比它所描述的堆要小四个数量级，所以访问可见性映射所需的物理 I/O 要少很多。在大部分情况下，可见性映射总是会被保留在内存中的缓冲中。

只用索引的扫描这章，比较有内容  [index-only-scan](http://www.postgres.cn/docs/11/indexes-index-only-scans.html)

