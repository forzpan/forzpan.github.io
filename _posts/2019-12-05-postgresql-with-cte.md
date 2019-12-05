---
title: PostgreSQL 之 WITH（公共表达式CTE）使用
date: 2019-12-05 02:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: 介绍PostgreSQL中WITH（公共表达式CTE）使用
---

# 概述

WITH语句是大型查询中使用的一种辅助语句，可以被看定义了只在一个查询中存在的临时表。

WITH子句中的每一个辅助语句可以是一个`SELECT`、`INSERT`、`UPDATE`或`DELETE`。

WITH子句本身也可以被附加到一个主语句，主语句也可以是`SELECT`、`INSERT`、`UPDATE`或`DELETE`。

一个demo：

```sql
WITH regional_sales AS (
        SELECT region, SUM(amount) AS total_sales
        FROM orders
        GROUP BY region
     ), top_regions AS (
        SELECT region
        FROM regional_sales
        WHERE total_sales > (SELECT SUM(total_sales)/10 FROM regional_sales)
     )
SELECT region,
       product,
       SUM(quantity) AS product_units,
       SUM(amount) AS product_sales
FROM orders
WHERE region IN (SELECT region FROM top_regions)
GROUP BY region, product;
```

# RECURSIVE关键字

可选的`RECURSIVE`修饰符将`WITH`从单纯的句法便利变成了一种在标准SQL中不能完成的特性。通过使用`RECURSIVE`，一个`WITH`查询可以引用它自己的输出。

计算从1到100的整数和的查询：

```sql
WITH RECURSIVE t(n) AS (
    VALUES (1)                               --非递归项
  UNION ALL
    SELECT n+1 FROM t WHERE n < 100          --递归项
)
SELECT sum(n) FROM t;
```

`t(n)`的`n`是结果表的列名，当列名确定时可以不写。

工作原理：
* （1）初始化结果表，非递归项放入临时表。
* （2）将临时表记录append到结果表（union的话，临时表记录在结果表不存在的会复制到结果表以外，已经存在的记录会被从临时表中剔除，处理完成后临时表为空则退出）。
* （3）拿临时表数据进行递归项计算，计算结果覆盖临时表，临时表为空则退出。
* （4）继续步骤（2）。

递归查询通常用于处理层次或者树状结构的数据，比如，组织部门表：

```sql
--建表
CREATE TABLE DEPARTMENT (
    PID INT,
    SID INT,
    SNAME VARCHAR
);
--入数据
INSERT INTO DEPARTMENT VALUES
(0,1,'A'),(0,2,'B'),(1,11,'AA'),(1,12,'AB'),(2,21,'BA'),(2,22,'BB'),
(11,111,'AAA'),(11,112,'AAB'),(12,121,'ABA'),(12,122,'ABB'),(21,211,'BAA'),(21,212,'BAB'),(22,221,'BBA'),(22,222,'BBB');
--查询
WITH RECURSIVE T AS (
    SELECT PID, SID, SNAME FROM DEPARTMENT WHERE PID = '0'
    UNION ALL
    SELECT P.PID, P.SID, P.SNAME FROM T, DEPARTMENT P WHERE P.PID = T.SID
)
SELECT * FROM T;
```

保证有向图是非常必要的，`环状`情况会导致`死循环`。结果集只关注`父项`和`子项`的查询可以使用`union`来清空临时表，正确退出递归查询。还有额外关注项目的情况下`union`不能消灭死循环。

举个栗子，使用link字段搜索graph表：

```sql
WITH RECURSIVE search_graph(id, link, data, depth) AS (
        SELECT g.id, g.link, g.data, 1
        FROM graph g
      UNION ALL
        SELECT g.id, g.link, g.data, sg.depth + 1
        FROM graph g, search_graph sg
        WHERE g.id = sg.link
)
SELECT * FROM search_graph;
```

如果`link->id->link`形成一个环，上面的SQL就陷入了死循环。改成`union`也没用，因为`depth`不一样，去不了重。

解决方案：增加路径数组字段path，利用它确定是否产生循环了，用cycle字段标记。

```sql
WITH RECURSIVE search_graph(id, link, data, depth, path, cycle) AS (
        SELECT g.id, g.link, g.data, 1,
          ARRAY[g.id],
          false
        FROM graph g
      UNION ALL
        SELECT g.id, g.link, g.data, sg.depth + 1,
          path || g.id,
          g.id = ANY(path)
        FROM graph g, search_graph sg
        WHERE g.id = sg.link AND NOT cycle
)
SELECT * FROM search_graph
WHERE NOT cycle;
```

可以看出来以上的递归是宽度优先遍历。

不确定查询是否可能循环时，一个测试查询的有用技巧是在父查询中放一个LIMIT。（相当于limit下推了）

```sql
WITH RECURSIVE t(n) AS (
    SELECT 1
  UNION ALL
    SELECT n+1 FROM t
)
SELECT n FROM t LIMIT 100;
```

因为PostgreSQL的实现只计算WITH查询中被父查询实际取到的行。不推荐在生产中使用这个技巧，不够会十分危险，外层查询排序递归查询的结果或者把它们连接成某种其他表，这个技巧将不会起作用。

# 优缺点

**优点**：避免查询中相同部分重复执行，提高效率。

**缺点**：筛选条件不能下推到with部分中，因为它会独立执行。

# WITH中的数据修改语句

```sql
WITH moved_rows AS (
    DELETE FROM products
    WHERE
        "date" >= '2010-10-01' AND
        "date" < '2010-11-01'
    RETURNING *
)
INSERT INTO products_log
SELECT * FROM moved_rows;
```

省去了显示事务，实现了数据移动。改动一下，是不行的，因为DML语句只能作为顶层WITH语句，不能放在附加在子查询上，普通的select类型with语句可以附加在子查询上。

```sql
INSERT INTO products_log
WITH moved_rows AS (
    DELETE FROM products
    WHERE
        "date" >= '2010-10-01' AND
        "date" < '2010-11-01'
    RETURNING *
)
SELECT * FROM moved_rows;
```

数据修改语句中不允许递归自引用。换句话说`WITH RECURSIVE`语句中不能使用DML语句。可以用一个递归with查出要处理的结果，再进一步处理：

```sql
WITH RECURSIVE included_parts(sub_part, part) AS (
    SELECT sub_part, part FROM parts WHERE part = 'our_product'
  UNION ALL
    SELECT p.sub_part, p.part
    FROM included_parts pr, parts p
    WHERE p.part = pr.sub_part
  )
DELETE FROM parts
  WHERE part IN (SELECT part FROM included_parts);
```

**WITH中的DML语句只被执行一次，并且总是能结束，不管主查询是否读取它们所有的输出。**

**WITH中SELECT语句，直到主查询要求SELECT的输出时，SELECT才会被执行。**

WITH中的子语句被和每一个其他子语句以及主查询并发执行。他们基于同一个snapshot执行。

```sql
WITH t AS (
    UPDATE products SET price = price * 1.05
    RETURNING *
)
SELECT * FROM products;
```

外层SELECT可以返回在UPDATE动作之前的原始价格，而在

```sql
WITH t AS (
    UPDATE products SET price = price * 1.05
    RETURNING *
)
SELECT * FROM t;
```

外部SELECT将返回更新过的数据。

试验了以下语句都只有外层主语句起作用：

```sql
with x as (update xxx set id = 1) update xxx set id = 2;

with x as (delete from xxx where id = 2) update xxx set id = 3;

with x as (update xxx set id = 4) delete from xxx where id = 3;
```

因此对文档中下面几句话还是比较疑惑的：

```
WITH中的子语句被和每一个其他子语句以及主查询并发执行。因此在使用WITH中的数据修改语句时，指定更新的顺序实际是以不可预测的方式发生的。
在一个语句中试图两次更新同一行是不被支持的。只会发生一次修改，但是该办法不能很容易地（有时是不可能）可靠地预测哪一个会被执行。
这也应用于删除一个已经在同一个语句中被更新过的行：只有更新被执行。
因此你通常应该避免尝试在一个语句中尝试两次修改同一个行。尤其是防止书写可能影响被主语句或兄弟子语句修改的相同行。这样一个语句的效果将是不可预测的。
```

在WITH中一个数据修改语句中被用作目标的任何表不能有条件规则、ALSO规则或INSTEAD规则，这些规则会扩展成为多个语句。

