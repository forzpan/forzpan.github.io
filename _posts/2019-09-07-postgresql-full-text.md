---
title: PostgreSQL 之 全文查询
date: 2019-09-07 05:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中全文查询相关内容
---

# 查询处理

```
  Schema   |    Name     | Result data type | Argument data types |  Type 
------------+-------------+------------------+---------------------+--------
 pg_catalog | to_tsvector | tsvector         | json                | normal
 pg_catalog | to_tsvector | tsvector         | jsonb               | normal
 pg_catalog | to_tsvector | tsvector         | regconfig, json     | normal
 pg_catalog | to_tsvector | tsvector         | regconfig, jsonb    | normal
 pg_catalog | to_tsvector | tsvector         | regconfig, text     | normal
 pg_catalog | to_tsvector | tsvector         | text                | normal
```

可以看出函数to_tsvector的几种用法，GIN索引中通过`CREATE INDEX pgweb_idx ON pgweb USING GIN(to_tsvector('english', body));`创建的索引。`WHERE to_tsvector('english', body) @@ 'a & b'`可以使用该索引，但`WHERE to_tsvector(body) @@ 'a & b'`不能。

可以建立更复杂的表达式索引，在其中配置名被另一个列指定，例如： 

```sql
CREATE INDEX pgweb_idx ON pgweb USING GIN(to_tsvector(config_name, body));
```

这里config_name是pgweb表中的一个列。这允许在同一个索引中有混合配置，同时记录哪个配置被用于每一个索引项。

索引甚至可以连接列：

```sql
CREATE INDEX pgweb_idx ON pgweb USING GIN(to_tsvector('english', title || ' ' || body));
```

函数to_tsquery、plainto_tsquery和phraseto_tsquery用来把一个查询转换成tsquery数据类型：

* 正规化

```sql
SELECT to_tsquery('english', 'The & Fat & Rats');
  to_tsquery  
---------------
 'fat' & 'rat'
```

* 附加权重

```sql
SELECT to_tsquery('english', 'Fat | Rats:AB');
    to_tsquery   
------------------
 'fat' | 'rat':AB
```

* 指定前缀匹配

```sql
SELECT to_tsquery('supern:*A & star:A*B');
        to_tsquery       
--------------------------
 'supern':*A & 'star':*AB
```

---

```sql
SELECT plainto_tsquery('english', 'The Fat Rats');
 plainto_tsquery
-----------------
 'fat' & 'rat'
```

* plainto_tsquery不会识别其输入中的tsquery操作符、权重标签或前缀匹配标签

```sql
SELECT plainto_tsquery('english', 'The Fat & Rats:C');
   plainto_tsquery  
---------------------
 'fat' & 'rat' & 'c'
```

* phraseto_tsquery的行为很像plainto_tsquery，不过前者会在留下来的词之间插入<->（FOLLOWED BY）操作符而不是&（AND）操作符。

```sql
SELECT phraseto_tsquery('english', 'The Fat Rats');
 phraseto_tsquery
------------------
 'fat' <-> 'rat'
```

* 和plainto_tsquery相似，phraseto_tsquery函数不会识别其输入中的tsquery操作符、权重标签或者前缀匹配标签

```sql
SELECT phraseto_tsquery('english', 'The Fat & Rats:C');
      phraseto_tsquery
-----------------------------
 'fat' <-> 'rat' <-> 'c'
```

* 停用词不是简单地丢弃掉，而是通过插入<N>操作符（而不是<->操作符）来解释。

```sql
SELECT phraseto_tsquery('english', 'The Fat and Rats');
 phraseto_tsquery
------------------
 'fat' <2> 'rat'
```

# 搜索排名

两种排名函数  ts_rank 和 ts_rank_cd，第一种是只考虑匹配词频，第二种还要考虑匹配词位相互之间的接近度（覆盖密度）。

```
ts_rank([ weights float4[], ] vector tsvector, query tsquery [, normalization integer ]) returns float4
ts_rank_cd([ weights float4[], ] vector tsvector, query tsquery [, normalization integer ]) returns float4
```

权重可以影响排名。权重数组指定每一类词应该得到多重的权重，按照如下的顺序：

`{D-权重, C-权重, B-权重, A-权重}`

如果没有提供权重，那么将使用这些默认值：

`{0.1, 0.2, 0.4, 1.0}`

文档越长，匹配机会越大，相关度肯定不能因为长就更相关。整数正规化选项处理这个问题：

* 0（默认值）忽略文档长度
* 1 用 1 + 文档长度的对数除排名
* 2 用文档长度除排名
* 4 用长度之间的平均调和距离除排名（只被ts_rank_cd实现）
* 8 用文档中唯一词的数量除排名
* 16 用 1 + 文档中唯一词数量的对数除排名
* 32 用排名 + 1 除排名

可以看出来是个可以进行位运算同时指定多个比如 2|4


# 高亮结果

```
ts_headline([ config regconfig, ] document text, query tsquery [, options text ]) returns text
```

# GIN 和 GiST 索引类型

有两种索引可以被用来加速全文搜索。

GIN 索引是更好的文本搜索索引类型。作为倒排索引，每个词（词位）在其中都有一个索引项，其中有压缩过的匹配位置的列表。多词搜索可以找到第一个匹配，然后使用该索引移除缺少额外词的行。GIN 索引只存储tsvector值的词（词位），并且不存储它们的权重标签。因此，在使用涉及权重的查询时需要一次在表行上的重新检查。

一个GiST索引是有损的，这表示索引可能产生假匹配，并且有必要检查真实的表行来消除这种假匹配（PostgreSQL在需要时会自动做这一步）。GiST 索引之所以是有损的，是因为每一个文档在索引中被表示为一个定长的签名。该签名通过哈希每一个词到一个n位串中的一个单一位来产生通过将所有这些位OR在一起产生一个n位的文档签名。当两个词哈希到同一个位位置时就会产生假匹配。如果查询中所有词都有匹配（真或假），则必须检索表行查看匹配是否正确。（类似布隆过滤器的办法）

注意GIN索引的构件时间常常可以通过增加[maintenance_work_mem](http://www.postgres.cn/docs/11/runtime-config-resource.html#GUC-MAINTENANCE-WORK-MEM)来改进，而GiST索引的构建时间则与该参数无关。


