---
title: PostgreSQL 之 系统列
date: 2019-09-18 01:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: PostgreSQL中系统列相关内容
---

# 系统列 

* `oid`

一行的对象标识符（对象ID）。该列只有在表使用WITH OIDS创建时或者default_with_oids 配置变量被设置为true时才存在。该列的类型为OID类型。

* `tableoid`

包含这一行所属表的OID。该列是特别为从继承层次中选择的查询而准备，因为如果没有它将很难知道一行来自于哪个表。tableoid可以与pg_class的oid列进行连接来获得表的名称。

* `xmin`

插入该行版本的事务身份（事务ID）。一个行版本是一个行的一个特别版本，对一个逻辑行的每一次更新都将创建一个新的行版本。

* `cmin`

插入事务中的命令序号（从0开始）。

* `xmax`

删除事务的身份（事务ID），对于未删除的行版本为0。对于一个可见的行版本，该列值也可能为非零。这通常表示删除事务还没有提交，或者一个删除尝试被回滚。

* `cmax`

删除事务中的命令序号，或者为0。

* `ctid`

行版本在其表中的物理位置。注意尽管ctid可以被用来快速地定位行版本，但行的ctid会在更新或VACUUM FULL移动时改变。因此，ctid不能作为长期行标识符。OID或用户定义的序列号才应该被用来标识逻辑行。

---

OID是32位量，它从一个服务于整个集簇的计数器分配而来。在一个大型的或者历时长久的数据库中，该计数器有可能会出现绕回。因此，不要总是假设OID是唯一的，除非你采取了措施来保证。

如果需要在一个表中标识行，推荐使用一个序列生成器。然而，OID也可以被使用，但是是要采取一些额外的预防措施：

* 如果要将OID用来标识行，应该在OID列上创建一个唯一约束。当这样一个唯一约束存在时，系统会注意不生成匹配现有行的OID（当然，这只有在表中行数少于2^32（40亿）时才成立。并且在实践中表尺寸最好远比这个值小，否则将会牺牲性能）。
* 绝不要认为OID在表之间也是唯一的，使用tableoid和行OID的组合来作为数据库范围内的标识符。
* 当然，问题中的表都必须是用WITH OIDS创建。

事务标识符也是32位量。在一个历时长久的数据库中事务ID同样会绕回。但如果采取适当的维护过程，这不会是一个致命的问题。但是，长期（超过10亿个事务）依赖事务ID的唯一性是不明智的。

命令标识符也是32位量。这对一个事务中包含的SQL命令设置了一个硬极限： 23^2（40亿）。在实践中，该限制并不是问题 — 注意该限制只是针对SQL命令的数目而不是被处理的行数。同样，只有真正修改了数据库内容的命令才会消耗一个命令标识符。

# 测试

```sql
-- 建表
create table test(id int, name varchar);
```

```sql
-- 插入
insert into test select generate_series(1,3), 'abc';
begin;
insert into test values (4, 'abc');
insert into test values (5, 'abc');
insert into test values (6, 'abc');
commit;
select xmin, cmin, xmax, cmax, ctid, * from test;
 
 xmin | cmin | xmax | cmax | ctid  | id | name
------+------+------+------+-------+----+------
  850 |    0 |    0 |    0 | (0,1) |  1 | abc
  850 |    0 |    0 |    0 | (0,2) |  2 | abc
  850 |    0 |    0 |    0 | (0,3) |  3 | abc
  851 |    0 |    0 |    0 | (0,4) |  4 | abc
  851 |    1 |    0 |    1 | (0,5) |  5 | abc
  851 |    2 |    0 |    2 | (0,6) |  6 | abc
```

可以看出850和851两个事务，同一个事务内部命令标识符从0开始递增，cmin和cmax这两个值基本都是一样的。


```sql
-- 更新
-- session1
begin;
update test set name = 'def' where id = 3;
select xmin, cmin, xmax, cmax, ctid, * from test;
 
 xmin | cmin | xmax | cmax | ctid  | id | name
------+------+------+------+-------+----+------
  850 |    0 |    0 |    0 | (0,1) |  1 | abc
  850 |    0 |    0 |    0 | (0,2) |  2 | abc
  851 |    0 |    0 |    0 | (0,4) |  4 | abc
  851 |    1 |    0 |    1 | (0,5) |  5 | abc
  851 |    2 |    0 |    2 | (0,6) |  6 | abc
  852 |    0 |    0 |    0 | (0,7) |  3 | def
                                                -- session2
                                                select xmin, cmin, xmax, cmax, ctid, * from test;
 
                                                 xmin | cmin | xmax | cmax | ctid  | id | name
                                                ------+------+------+------+-------+----+------
                                                  850 |    0 |    0 |    0 | (0,1) |  1 | abc
                                                  850 |    0 |    0 |    0 | (0,2) |  2 | abc
                                                  850 |    0 |  852 |    0 | (0,3) |  3 | abc
                                                  851 |    0 |    0 |    0 | (0,4) |  4 | abc
                                                  851 |    1 |    0 |    1 | (0,5) |  5 | abc
                                                  851 |    2 |    0 |    2 | (0,6) |  6 | abc
rollback;
select xmin, cmin, xmax, cmax, ctid, * from test;
 
 xmin | cmin | xmax | cmax | ctid  | id | name
------+------+------+------+-------+----+------
  850 |    0 |    0 |    0 | (0,1) |  1 | abc
  850 |    0 |    0 |    0 | (0,2) |  2 | abc
  850 |    0 |  852 |    0 | (0,3) |  3 | abc
  851 |    0 |    0 |    0 | (0,4) |  4 | abc
  851 |    1 |    0 |    1 | (0,5) |  5 | abc
  851 |    2 |    0 |    2 | (0,6) |  6 | abc
```

可以看出更新之后原来的行xmax被打上新的事务id 852 ，同时新增一行记录 事务号也是852，rollback之后 xmax的值还是存在的。

```sql
-- session1
begin;
update test set name = 'xyz' where id = 3;
update test set name = 'xyz' where id = 2;
select xmin, cmin, xmax, cmax, ctid, * from test;
 
 xmin | cmin | xmax | cmax | ctid  | id | name
------+------+------+------+-------+----+------
  850 |    0 |    0 |    0 | (0,1) |  1 | abc
  851 |    0 |    0 |    0 | (0,4) |  4 | abc
  851 |    1 |    0 |    1 | (0,5) |  5 | abc
  851 |    2 |    0 |    2 | (0,6) |  6 | abc
  854 |    0 |    0 |    0 | (0,8) |  3 | xyz
  854 |    1 |    0 |    1 | (0,9) |  2 | xyz
                                                -- session2
                                                select xmin, cmin, xmax, cmax, ctid, * from test;
 
                                                 xmin | cmin | xmax | cmax | ctid  | id | name
                                                ------+------+------+------+-------+----+------
                                                  850 |    0 |    0 |    0 | (0,1) |  1 | abc
                                                  850 |    1 |  854 |    1 | (0,2) |  2 | abc
                                                  850 |    0 |  854 |    0 | (0,3) |  3 | abc
                                                  851 |    0 |    0 |    0 | (0,4) |  4 | abc
                                                  851 |    1 |    0 |    1 | (0,5) |  5 | abc
                                                  851 |    2 |    0 |    2 | (0,6) |  6 | abc
commit;
```

可以看出，853被消耗之后，不再重用，直接使用854，标记两条旧记录的xmax，新增两条854的记录。ctid(0,2)(0,3)(0,7)对应的记录最终会被vacuum。

```sql
-- 删除
-- session1
begin;
delete from test where id = 6;
delete from test where id = 5;
                                                -- session2
                                                select xmin, cmin, xmax, cmax, ctid, * from test;
 
                                                 xmin | cmin | xmax | cmax | ctid  | id | name
                                                ------+------+------+------+-------+----+------
                                                  850 |    0 |    0 |    0 | (0,1) |  1 | abc
                                                  851 |    0 |    0 |    0 | (0,4) |  4 | abc
                                                  851 |    1 |  855 |    1 | (0,5) |  5 | abc
                                                  851 |    0 |  855 |    0 | (0,6) |  6 | abc
                                                  854 |    0 |    0 |    0 | (0,8) |  3 | xyz
                                                  854 |    1 |    0 |    1 | (0,9) |  2 | xyz
commit;
select xmin, cmin, xmax, cmax, ctid, * from test;
 
 xmin | cmin | xmax | cmax | ctid  | id | name
------+------+------+------+-------+----+------
  850 |    0 |    0 |    0 | (0,1) |  1 | abc
  851 |    0 |    0 |    0 | (0,4) |  4 | abc
  854 |    0 |    0 |    0 | (0,8) |  3 | xyz
  854 |    1 |    0 |    1 | (0,9) |  2 | xyz
```

可以看出删除和update中有相似之处，旧记录xmax打标。

可以看出上面所有的cmin和cmax总是相同的，事实也是，他们就是一个东西，文档太旧了而已。

如果了解MVCC，上面的测试应该很好懂。

补充：

SAVEPOINT会产生一个新的子事务号码XMIN或者XMAX，但是事务中CMIN , CMAX值是连续的，不会因为新事务重置。

VACUUM会更新了FSM信息。让页面重用，新插入的数据CTID字段不一定大。


