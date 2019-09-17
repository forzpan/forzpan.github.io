---
title: PostgreSQL 之 窗口函数
date: 2019-09-17 01:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: PostgreSQL中窗口函数相关内容
---

窗口函数可以很好的避免一些语法上的关联操作，简化了sql的写法。

样例表：

```sql
pgdba->pgdba@[local]=# SELECT * FROM empsalary;
  depname  | empno | salary
-----------+-------+--------
 develop   |    11 |   5200
 develop   |     7 |   4200
 develop   |     9 |   4500
 develop   |     8 |   6000
 develop   |    10 |   5200
 develop   |     6 |
 personnel |     5 |   3500
 personnel |     2 |   3900
 sales     |     3 |   4800
 sales     |     1 |   5000
 sales     |     4 |   4800
(11 rows)
```

# 初识窗口函数

* 部门，工号，薪资，部门平均薪资对比。

```sql
pgdba->pgdba@[local]=# SELECT depname, empno, salary, avg(salary) OVER (PARTITION BY depname) FROM empsalary;
  depname  | empno | salary |          avg
-----------+-------+--------+-----------------------
 develop   |    11 |   5200 | 5020.0000000000000000
 develop   |     7 |   4200 | 5020.0000000000000000
 develop   |     9 |   4500 | 5020.0000000000000000
 develop   |     8 |   6000 | 5020.0000000000000000
 develop   |    10 |   5200 | 5020.0000000000000000
 develop   |     6 |        | 5020.0000000000000000
 personnel |     5 |   3500 | 3700.0000000000000000
 personnel |     2 |   3900 | 3700.0000000000000000
 sales     |     3 |   4800 | 4866.6666666666666667
 sales     |     1 |   5000 | 4866.6666666666666667
 sales     |     4 |   4800 | 4866.6666666666666667
(11 rows)
```

* 部门，工号，薪资，部门内部按薪资排序。

```sql
pgdba->pgdba@[local]=# SELECT depname, empno, salary, rank() OVER (PARTITION BY depname ORDER BY salary DESC NULLS LAST) FROM empsalary;
  depname  | empno | salary | rank
-----------+-------+--------+------
 develop   |     8 |   6000 |    1
 develop   |    10 |   5200 |    2
 develop   |    11 |   5200 |    2
 develop   |     9 |   4500 |    4
 develop   |     7 |   4200 |    5
 develop   |     6 |        |    6
 personnel |     2 |   3900 |    1
 personnel |     5 |   3500 |    2
 sales     |     1 |   5000 |    1
 sales     |     4 |   4800 |    2
 sales     |     3 |   4800 |    2
(11 rows)
```

* WINDOW子句

当查询涉及到多个窗口函数时，可以将每一个分别写在一个独立的OVER子句中。当多个函数要求同一窗口时，这种做法是冗余易错。可以用一个命名的WINDOW子句定义窗口行为，然后在OVER中引用它。

```sql
pgdba->pgdba@[local]=# SELECT depname, empno, salary, sum(salary) OVER w, avg(salary) OVER w FROM empsalary WINDOW w AS (PARTITION BY depname ORDER BY salary DESC);
  depname  | empno | salary |  sum  |          avg
-----------+-------+--------+-------+-----------------------
 develop   |     6 |        |       |
 develop   |     8 |   6000 |  6000 | 6000.0000000000000000
 develop   |    11 |   5200 | 16400 | 5466.6666666666666667
 develop   |    10 |   5200 | 16400 | 5466.6666666666666667
 develop   |     9 |   4500 | 20900 | 5225.0000000000000000
 develop   |     7 |   4200 | 25100 | 5020.0000000000000000
 personnel |     2 |   3900 |  3900 | 3900.0000000000000000
 personnel |     5 |   3500 |  7400 | 3700.0000000000000000
 sales     |     1 |   5000 |  5000 | 5000.6666666666666667
 sales     |     4 |   4800 | 14600 | 4866.6666666666666667
 sales     |     3 |   4800 | 14600 | 4866.6666666666666667
(11 rows)
```

* 总结一下：

一个窗口函数所作用的行集是通过FROM子句产生，再经过WHERE、GROUP BY、HAVING过滤得到的`虚拟表`。例如，一个由于不满足WHERE条件被删除的行是不会被任何窗口函数所见的。

一个查询中可包含多个窗口函数，每个窗口函数都可以用不同的OVER子句来按不同方式划分数据，但是它们都作用在由虚拟表定义的同一个行集上。

当多个窗口函数被使用时，窗口定义中所有等效的PARTITION BY、ORDER BY子句的窗口函数被保证在数据上的同一趟扫描中计算。因此它们将会看到相同的排序顺序，即使没有明确的ORDER BY来指定一个顺序。

```sql
-- 两个窗口函数在一趟计算中完成。
SELECT depname
      ,empno
      ,salary
      ,sum(salary) OVER (PARTITION BY depname ORDER BY salary DESC)
      ,avg(salary) OVER (PARTITION BY depname ORDER BY salary DESC)
from empsalary;

-- 即使没有明确的ORDER BY指定顺序。
SELECT depname
      ,empno
      ,salary
      ,sum(salary) OVER (PARTITION BY depname)
      ,avg(salary) OVER (PARTITION BY depname)
from empsalary;
```

但是，对于具有不同PARTITION BY、ORDER BY定义的函数，计算没有这种保证（在这种情况中，在多个窗口函数计算之间通常要求一个排序步骤，并且并不保证保留行的顺序，即使它的ORDER BY把这些行视为等效的）。

```sql
-- 两个ORDER BY不同，串行执行。
SELECT depname
      ,empno
      , salary
      , sum(salary) OVER (PARTITION BY depname ORDER BY salary DESC)
      , avg(salary) OVER (PARTITION BY depname ORDER BY salary)
from empsalary;
```

可以用EXPALIN查看：

```
                                    QUERY PLAN
--------------------------------------------------------------------------------------
  WindowAgg  (cost=119.63..136.83 rows=860 width=106)
    ->  Sort  (cost=119.63..121.78 rows=860 width=74)
          Sort Key: depname, salary
          ->  WindowAgg  (cost=60.52..77.72 rows=860 width=74)
                ->  Sort  (cost=60.52..62.67 rows=860 width=66)
                      Sort Key: depname, salary DESC
                      ->  Seq Scan on empsalary  (cost=0.00..18.60 rows=860 width=66)
```

当然，如果行的顺序不重要时ORDER BY可以忽略。PARTITION BY同样也可以被忽略，在这种情况下只会产生一个包含所有行的分区。

换而言之，在OVER子句中没有ORDER BY，窗口帧和分区一样，而如果再缺少PARTITION BY则和整个虚拟表一样。

到这里可能会有`分区`和`窗口帧`的概念疑惑。窗口函数的分区好理解，就是由PARTITION BY指定的。窗口帧下面讨论。。。

# 窗口函数语法详解

一个窗口函数调用的语法是下列之一：

```
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
```

其中`window_definition`的语法是：

```
[ existing_window_name ]
[ PARTITION BY expression [, ...] ]
[ ORDER BY expression [ ASC | DESC | USING operator ] [ NULLS { FIRST | LAST } ] [, ...] ]
[ frame_clause ]
```

而可选的`frame_clause`是下列之一：

```
{ RANGE | ROWS } frame_start
{ RANGE | ROWS } BETWEEN frame_start AND frame_end
```

其中`frame_start`和`frame_end`可以是下面形式中的一种

```
UNBOUNDED PRECEDING
value PRECEDING  （RANGE不可用）
CURRENT ROW
value FOLLOWING  （RANGE不可用）
UNBOUNDED FOLLOWING
```

**名词解释**：

* FILTER子句，那么只有对filter_clause计算为真的输入行会被交给该窗口函数，其他行会被丢弃。只有是聚合的窗口函数才接受FILTER ，rank()这种就不接受。
* expression 表示任何自身不含有窗口函数调用的值表达式。
* window_name 是对定义在查询的WINDOW子句中的一个命名窗口声明的引用。OVER window_name并不严格地等价于OVER (window_name ...)，后者表示复制并修改窗口定义，并且在被引用窗口声明包括一个frame_clause时会被拒绝。
* PARTITION BY子句将查询的行分组成为`分区`，窗口函数会独立地处理它们。
* ORDER BY子句决定被窗口函数处理的一个分区中的行的顺序。如果没有ORDER BY，行将被以未指定的顺序被处理。
* frame_clause指定构成`窗口帧`的行集，是当前分区的子集，窗口函数作用在该帧而非整个分区。帧被指定为RANGE或ROWS模式，在两种情况中它都从frame_start运行到frame_end。如果frame_end被忽略，默认运行到CURRENT ROW。
    * frame_start为 UNBOUNDED PRECEDING 表示该帧开始于分区的第一行。
    * frame_end为 UNBOUNDED FOLLOWING 表示该帧结束于分区的最后一行。
    * 在RANGE模式下， frame_start为CURRENT ROW表示该帧开始于当前行的第一个平级行（被ORDER BY认为与当前行等效的行），而frame_end为CURRENT ROW表示该帧结束于当前行的最后一个平级行。
    * 在ROWS模式下，CURRENT ROW仅表示当前行。
    * value PRECEDING 和 value FOLLOWING 目前只在ROWS模式中被允许。它们指示帧开始或结束于当前行之前或之后的指定数量的行。
    * value 必须是一个不包含任何变量、聚合函数或窗口函数的整数表达式，该值不能为空或负，但是可以为零，零表示只选择当前行。

`默认的帧`选项是RANGE UNBOUNDED PRECEDING，它和RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW效果相同。

如果使用ORDER BY，这会把该帧设置为从分区开始一直到当前行的最后一个ORDER BY平级行的所有行。

如果不使用ORDER BY，分区中所有的行都被包括在窗口帧中，因为所有行都成为了当前行的平级行。

**限制**：

frame_start不能为UNBOUNDED FOLLOWING、frame_end不能为UNBOUNDED PRECEDING

frame_end的选择不能早于frame_start的选择出现 ，例：RANGE BETWEEN CURRENT ROW AND value PRECEDING 是不被允许的。

使用\*的语法被用来把参数较少的聚合函数当作窗口函数调用， 例如count(\*) OVER (PARTITION BY x ORDER BY y)。 星号（*）通常通常不用于窗口特定的函数。窗口特定的函数不允许在函数参数列表中使用DISTINCT或ORDER BY。

只有在SELECT列表和查询的ORDER BY子句中才允许窗口函数调用。

**写个完整一点的例子**：

```sql
SELECT depname
      ,empno
      ,salary
      ,array_agg(salary) FILTER (WHERE depname='develop') OVER (PARTITION BY depname order by salary NULLS FIRST ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)
FROM empsalary;
```

# 通用窗口函数

函数|返回类型|描述
----|-------|-----
row_number()|bigint|当前行在其分区中的行号，从1计
rank()|bigint|带间隙的当前行排名； 与该行的第一个同等行的row_number相同
dense_rank()|bigint|不带间隙的当前行排名； 这个函数计数同等组
percent_rank()|double precision|当前行的相对排名： (rank- 1) / (总行数 - 1)
cume_dist()|double precision|累积分布：(在当前行之前或者平级的分区行数) / 分区行总数
ntile(num_buckets integer)|integer|从1到参数值的整数范围，尽可能等分分区
lag(value anyelement [, offset integer [, default anyelement ]])|和value的类型相同|返回value，它在分区内当前行的之前offset个位置的行上计算；如果没有这样的行，返回default替代（必须和value类型相同）。offset和default都是根据当前行计算的结果。如果忽略它们，则offset默认是1，default默认是空值
lead(value anyelement [, offset integer [, default anyelement ]])|和value类型相同|返回value，它在分区内当前行的之后offset个位置的行上计算；如果没有这样的行，返回default替代（必须和value类型相同）。offset和default都是根据当前行计算的结果。如果忽略它们，则offset默认是1，default默认是空值
first_value(value any)|和value的类型相同|返回在窗口帧中第一行上计算的value
last_value(value any)|和value类型相同|返回在窗口帧中最后一行上计算的value
nth_value(value any, nth integer)|和value类型相同|返回在窗口帧中第nth行（行从1计数）上计算的value；没有这样的行则返回空值

除了这些函数外，任何内建的或用户定义的通用或统计聚集（也就是有序集或假想集聚集除外）都可以作为窗口函数，仅当调用跟着OVER子句时，聚集函数才会作为窗口函数。

# 相关链接

[窗口函数](http://www.postgres.cn/docs/11/tutorial-window.html)

[窗口函数调用](http://www.postgres.cn/docs/11/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)

[通用窗口函数](http://www.postgres.cn/docs/11/functions-window.html#FUNCTIONS-WINDOW-TABLE)

[PostgreSQL的window函数应用整理](https://my.oschina.net/Kenyon/blog/79543)




