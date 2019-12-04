---
title: PostgreSQL 之 一些SQL
date: 2019-12-04 02:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: PostgreSQL中一些SQL的语法
---

# LATERAL子查询

```sql
SELECT * FROM foo, LATERAL (SELECT * FROM bar WHERE bar.id = foo.bar_id) ss
```

本来SELECT中多个子查询拼接的列，这边一下搞定。

# values生成常量表

```sql
SELECT * FROM (VALUES (1, 'one'), (1, 'one'), (3, 'three')) AS t (num,letter);
```

# 部分distinct

```sql
SELECT DISTINCT ON (expression [, expression ...]) select_list ...
```

# GROUPING SETS、ROLLUP 和 CUBE

```sql
=> SELECT * FROM items_sold;
brand | size | sales
------+------+-------
  Foo |  L   | 10
  Foo |  M   | 20
  Bar |  M   | 15
  Bar |  L   | 5
(4 rows)

=> SELECT brand, size, sum(sales) FROM items_sold GROUP BY GROUPING SETS ((brand), (size), ());
brand | size | sum
------+------+-----
  Foo |      | 30
  Bar |      | 20
      |  L   | 15
      |  M   | 35
      |      | 50
(5 rows)
```

```sql
ROLLUP ( e1, e2, e3, ... )
-- 等效于
GROUPING SETS (
　　( e1, e2, e3, ... ),
...
( e1, e2 ),
( e1 ),
( )
)
```

```sql
CUBE ( a, b, c )
-- 等效于
GROUPING SETS (
( a, b, c ),
( a, b ),
( a, c ),
( a ),
( b, c ),
( b ),
( c ),
( )
)
```

```sql
CUBE ( (a, b), (c, d) )
-- 等效于
GROUPING SETS (
( a, b, c, d ),
( a, b ),
( c, d ),
( )
)
```

```sql
GROUP BY a, CUBE (b, c), GROUPING SETS ((d), (e))
-- 等效于
GROUP BY GROUPING SETS (
(a, b, c, d), (a, b, c, e),
(a, b, d), (a, b, e),
(a, c, d), (a, c, e),
(a, d), (a, e)
)
```

