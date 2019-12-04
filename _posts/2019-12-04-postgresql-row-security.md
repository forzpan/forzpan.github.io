---
title: PostgreSQL 之 行安全性策略
date: 2019-12-04 05:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: 介绍PostgreSQL中行安全性策略
---

# 介绍

使用[ALTER TABLE ... ENABLE ROW LEVEL SECURITY](http://www.postgres.cn/docs/10/sql-altertable.html)在一个表上启用行安全性 。

表拥有者通常也能绕过行安全性，不过表拥有者可以选择用`ALTER TABLE ... FORCE ROW LEVEL SECURITY`来服从行安全性。

具有BYPASSRLS属性的超级用户和角色在访问一个表时总是可以绕过行安全性系统。

一旦启用，所有对该表行进行选择或者修改的普通操作都必须被一条行安全性策略所允许。

如果表上不存在策略，将使用一条默认的否定策略，即所有的行都不可见或者不能 被修改。

但是应用在整个表上的操作不服从行安全性，例如`TRUNCATE`和`REFERENCES`。

行安全性策略可以针对特定的命令、角色或者两者。一条策略可以被指定为 适用于ALL命令，或者SELECT、 INSERT、UPDATE或者DELETE。 可以为一条给定策略分配多个角色，并且通常的角色成员关系和继承规则也适用。

对于每一行，在计算任何来自用户查询的条件或函数之前，先会计算这个策略是否通过。

当多条策略适用于一个给定查询时，会用OR（用于宽松的策略，这是默认的）或AND（用于限制性策略）将它们组合起来。多个限制性策略必须都通过，然后至少有一个宽松策略，还得通过。

# 语法详解

```sql
CREATE POLICY name ON table_name
    [ AS { PERMISSIVE | RESTRICTIVE } ]                                     -- 宽松的 或 严格的。默认是 宽容的
    [ FOR { ALL | SELECT | INSERT | UPDATE | DELETE } ]                     -- 允许哪些权限，默认是ALL，所有权限。未被指定的权限直接否定
    [ TO { role_name | PUBLIC | CURRENT_USER | SESSION_USER } [, ...] ]     -- 角色，默认是PUBLIC
    [ USING ( using_expression ) ]                                          -- 已经存在的行的检查条件，（with check不存在，使用using检查更新后的行）。insert策略不能有这个
    [ WITH CHECK ( check_expression ) ]                                     -- 新增行的检查条件，这理不止insert，由于mvcc，update也新增行。select策略和delete策略不能有这个
```

如果没有定义WITH CHECK表达式，那么 USING表达式将被用于决定哪些行可见（普通 USING情况）以及允许哪些行被增加（WITH CHECK情况）。

update操作需要select和update策略

delete操作需要select和delete策略

# 例子

作为一个简单的例子，这里是如何在account关系上 创建一条策略以允许只有managers角色的成员能访问行， 并且只能访问它们账户的行：

```sql
CREATE TABLE accounts (manager text, company text, contact_email text);
 
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
 
CREATE POLICY account_managers ON accounts TO managers USING (manager = current_user);
```

如果没有指定角色或者使用了特殊的用户名PUBLIC， 则该策略适用于系统上所有的用户。要允许所有用户访问users 表中属于他们自己的行，可以使用一条简单的策略：

```sql
CREATE POLICY user_policy ON users USING (user_name = current_user);
```

要对相对于可见行是被增加到表中的行使用一条不同的策略，可以使用 WITH CHECK子句。

这条策略将允许所有用户查看 users表中的所有行，但是只能修改它们自己的行（实际可以删除其他user的行）：

```sql
CREATE POLICY user_policy ON users USING (true) WITH CHECK (user_name = current_user);
```

