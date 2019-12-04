---
title: PostgreSQL 之 角色
date: 2019-12-04 04:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: 介绍PostgreSQL中角色
---

数据库角色在概念上已经完全与操作系统用户独立开来。

数据库角色在一个数据库集簇安装范围内是全局的（而不是独立数据库内）。

role和user几乎一样。

```
CREATE USER name  ==  CREATE ROLE name LOGIN;
```

* 查看角色使用

```sql
=> SELECT rolname FROM  pg_roles;
-- 或者
=> \du
```

* 角色有相关属性：`login privilege`，`superuser status`，`database creation`，`role creation`，`initiating replication`，`password`

```sql
CREATE ROLE name LOGIN                      -- 创建一个带有登录权限的角色
CREATE ROLE name SUPERUSER                  -- 创建一个超级用户权限的角色
CREATE ROLE name CREATEDB                   -- 创建一个带有创建数据库权限的角色
CREATE ROLE name CREATEROLE                 -- 创建一个带有创建普通角色权限的角色
CREATE ROLE name REPLICATION LOGIN          -- 创建一个带有流复制权限的角色
CREATE ROLE name PASSWORD 'string'          -- 创建一个带有密码验证的角色
```

```sql
CREATE ROLE name [ [ WITH ] option [ ... ] ]
 
其中option可以是：
      SUPERUSER | NOSUPERUSER
    | CREATEDB | NOCREATEDB
    | CREATEROLE | NOCREATEROLE
    | INHERIT | NOINHERIT
    | LOGIN | NOLOGIN
    | REPLICATION | NOREPLICATION
    | BYPASSRLS | NOBYPASSRLS
    | CONNECTION LIMIT connlimit
    | PASSWORD 'password'
    | VALID UNTIL 'timestamp'
    | IN ROLE role_name [, ...]
    | ROLE role_name [, ...]
    | ADMIN role_name [, ...]
```

一个角色也可以有角色相关的默认值，例如，如果出于某些原因你希望在每次连接时禁用索引扫描：

```sql
ALTER ROLE myname SET enable_indexscan TO off;   -- 当前session不生效，后续session生效
SET enable_indexscan TO off                      -- 当前session临时生效
ALTER ROLE rolename RESET enable_indexscan       -- 重置配置
```

* 角色的赋予与移除

```sql
GRANT group_role TO role1, ... ;
 
REVOKE group_role FROM role1, ... ;
```

不允许环状，不允许将一个角色授予给PUBLIC

* 角色权限的继承

```sql
CREATE ROLE joe LOGIN INHERIT;      -- 能够继承授予它的角色的权限
CREATE ROLE admin NOINHERIT;        -- 不能够继承授予它的角色的权限
CREATE ROLE wheel NOINHERIT;        -- 不能够继承授予它的角色的权限
GRANT admin TO joe;                 -- joe直接继承了admin的权限
GRANT wheel TO admin;               -- admin没有直接继承wheel 的权限
```

joe通过    SET ROLE admin;   和  SET ROLE wheel ; 可以直接变身为它们，以它们的身份行事

角色属性LOGIN、SUPERUSER、CREATEDB和CREATEROLE可以被认为是一种特殊权限，但是它们从来不会像数据库对象上的普通权限那样被继承

* 销毁角色

```sql
DROP ROLE name；
```

销毁一个角色，往往还需要删除或者转移该角色拥有的对象

```sql
ALTER TABLE bobs_table OWNER TO alice;
```

`REASSIGN OWNED` 和 `DROP OWNED` 整体转移和删除

移除曾经拥有过对象的角色的方法是

```sql
REASSIGN OWNED BY doomed_role TO successor_role;
DROP OWNED BY doomed_role;
-- 在集簇中的每一个数据库中重复上述命令
DROP ROLE doomed_role;
```

