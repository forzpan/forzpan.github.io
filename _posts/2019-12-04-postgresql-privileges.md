---
title: PostgreSQL 之 权限
date: 2019-12-04 06:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: 介绍PostgreSQL中相关权限
---

# SELECT

* 这个特权允许从指定表、视图或序列的任何列或者列出的特定列进行`SELECT`。
* 还允许使用`COPY TO`。
* 在`UPDATE`或`DELETE`中引用已有列值时也需要这个特权。
* 对于序列，这个特权也允许使用`currval`函数。
* 对于大对象，这个特权允许读取对象。

# INSERT

* 这个特权允许`INSERT`一个新行到指定表中。
* 如果列出了特定的列，只有这些列能在`INSERT`命令中被赋值（其他列将因此收到默认值）。
* 还允许`COPY FROM`。

# UPDATE

* 允许对指定表、视图或序列的任何列或者列出的特定列进行`UPDATE`（实际上，任何非平凡的`UPDATE`命令也会要求`SELECT`特权，因为它必须引用表列来判断哪些行要被更新或者为列计算新值）。
* 除SELECT特权之外，`SELECT ... FOR UPDATE`以及`SELECT ... FOR SHARE`也要求至少一列上的这个特权。
* 对于序列，这个特权允许使用`nextval`和`setval`函数。
* 对于大对象，这个特权允许写入或者截断对象。

# DELETE

* 允许从指定的表中`DELETE`一行（实际上，任何非平凡的`DELETE`命令也将要求`SELECT`特权，因为它必须引用表列来判断要删除哪些行）。

# TRUNCATE

* 允许在指定的表上`TRUNCATE`。

# REFERENCES

* 允许创建引用指定表或指定表列的外键约束。（请参阅[CREATE TABLE](http://www.postgres.cn/docs/10/sql-createtable.html)语句）。

# TRIGGER

* 允许在指定的表上创建触发器（见[CREATE TRIGGER](http://www.postgres.cn/docs/10/sql-createtrigger.html)语句）。

# CREATE

* 对于数据库，允许在其中创建新模式或发布。
* 对于模式，允许在其中创建新的对象。要重命名一个已有对象，你必须拥有该对象并且具有所在模式的这个特权。
* 对于表空间，允许在其中创建表、索引和临时文件，并且允许创建使用该表空间作为默认表空间的数据库（注意撤回这个特权将不会更改现有对象的放置位置）。

# CONNECT

* 允许用户连接到指定数据库。在连接开始时会检查这个特权（除了检查由`pg_hba.conf`施加的任何限制之外）。

# TEMPORARY/TEMP

* 允许在使用指定数据库时创建临时表。

# EXECUTE

* 允许使用指定的函数以及使用在该函数之上实现的任何操作符。这是适用于函数的唯一一种特权类型（这种语法也可用于聚集函数）。

# USAGE

* 对于过程语言，允许使用指定的语言创建函数。这是适用于过程语言的唯一一种特权类型。
* 对于模式，允许访问包含在指定模式中的对象（假定这些对象的拥有特权要求也满足）。本质上这允许被授权者在模式中“查阅”对象。如果没有这个权限，还是有可能看到对象名称，例如通过查询系统表。还有，在撤回这个权限之后，现有后端可能有语句之前已经执行过这种查阅，因此这不是一种阻止对象访问的完全安全的方法。
* 对于序列，这种特权允许使用`currval`和`nextval`函数。
* 对于类型和域，这种特权允许用该类型或域来创建表、函数和其他模式对象（注意这不能控制类型的一般“用法”，例如出现在查询中的该类型的值。它只阻止基于该类型创建对象。该特权的主要目的是控制哪些用户在一个类型上创建了依赖，这能够阻止拥有者以后更改该类型）。
* 对于外部数据包装器，这个特权允许使用该外部数据包装器创建新服务器。
* 对于服务器，这个特权允许使用该服务器创建外部表， 被授权者还可以创建、修改或删除与该服务器相关的他们自己的用户映射。

---

换个角度：

* 对于数据库，有3种权限`CTc`就是`Create`、`Temp`和`connect`三种，一个新建的数据库，`public`用户拥有`Tc`权限，`owner`拥有`CTc`权限。`revoke all on database mydb from PUBLIC;`可以移除这些权限。
* 对于模式，有两种权限`UC`就是`USAGE`和`CREATE`。默认只有拥有者有这些权限，`public`用户没有任何权限。每个`db`从模板复制来的`public`模式，在模版里`public`模式属于`postgres`，给与`PUBLIC`用户`UC`权限，所以任意用户都可以在这个模式创建表等一些对象。
* 对于表，有七种权限`arwdDxt`就是`insert`、`select`、`update`、`delete`、`truncate`、`references`和`trigger`。默认只有拥有者有这些权限，`public`用户没有任何权限。
* 对于列，有四种权限`arwx`就是`select`、`insert`、`update`和`references`。
* 对于序列，有三种权限`Urw`就是`select`、`update`和`usage`。


# 其他内容

```sql
=> \dp mytable
                              Access privileges
 Schema |  Name   | Type  |   Access privileges   | Column access privileges
--------+---------+-------+-----------------------+--------------------------
 public | mytable | table | miriam=arwdDxt/miriam | col1:
                          : =r/miriam             :   miriam_rw=rw/miriam
                          : admin=arw/miriam
--说明
角色名=xxxx -- 被授予给一个角色的特权
=xxxx      -- 被授予给 PUBLIC 的特权
        r  -- SELECT ("读")
        w  -- UPDATE ("写")
        a  -- INSERT ("追加")
        d  -- DELETE
        D  -- TRUNCATE
        x  -- REFERENCES
        t  -- TRIGGER
        X  -- EXECUTE
        U  -- USAGE
        C  -- CREATE
        c  -- CONNECT
        T  -- TEMPORARY
  arwdDxt  -- ALL PRIVILEGES （对于表，对其他对象会变化）
        *  -- 用于前述特权的授权选项

/yyyy -- 授予该特权的角色
```

* 角色的`admin option`特性

```sql
CREATE user user0;                              -- 创建用户 user0
CREATE user user1;                              -- 创建用户 user1
CREATE role role0;                              -- 创建角色 role0
set role role0;                                 -- 切换role到 role0
grant role0 to user1;                           -- 不能成功分配角色失败
```

总结，role自己不持有自身的`ADMIN OPTION`权限。

```sql
CREATE user user0;                              -- 创建用户 user0
CREATE user user1;                              -- 创建用户 user1
CREATE role role0 admin user0;                  -- 创建角色 role0，指定admin角色user0
set role user0;                                 -- 切换role到 user0
grant role0 to user1;                           -- 成功分配角色
grant role0 to user1 with admin option;         -- 成功分配角色，并赋予admin option权限
set role user1;                                 -- 切换role到 user1
revoke admin option for role0 from user0;       -- 从user0回收的role0的admin option权限
revoke admin option for role0 from user1;       -- 从user1回收的role0的admin option权限
revoke role0 from user0;                        -- 从user0回收的role0角色
```

总结
* 带有`admin option`的`grant`语句会覆盖不带`admin option`的`grant`语句。
* 带有`admin option`权限的`member role`可以增加新的`member role`，甚至新的`member role`也可以带`admin option`权限。
* 带有`admin option`权限的`member role`，可以互相撤销彼此`admin option`权限和成员资格。

```sql
CREATE user user0;                              -- 创建用户 user0
CREATE role role0 role user0;                   -- 创建角色 role0 指定member role
CREATE user user1 in role role0;                -- 创建用户 user1
CREATE user user2;                              -- 创建用户 user2
set role role0;                                 -- 切换role到 role0
create table test(id int);                      -- 创建表test，表拥有者是role0
set role user0;                                 -- 切换role到 role0
grant select on test TO user2;                  -- user0使用role0的身份去分配test表的select权限给user2
set role user1;                                 -- 切换role到 role0
revoke select on test from user2;               -- user1使用role0的身份去从user2回收test表的select权限
```

总结，`member role`可以使用`父role`的角色去对表等对象执行`grant`和`revoke`操作。

另外，对于角色的`revoke`语法中`cascade`好像没有什么用处。回收一个表上的权限会回收所有列上的相同权限。

