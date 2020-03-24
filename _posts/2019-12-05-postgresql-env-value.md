---
title: PostgreSQL 之 环境变量
date: 2019-12-05 05:00:00
categories:
- PostgreSQL-Administrate
tags:
- PostgreSQL
description: 介绍PostgreSQL中的环境变量
---


`PGHOST`（主机名） 或 `PGHOSTADDR`（主机ip地址）

`PGPORT`

`PGDATABASE`

`PGUSER`

`PGPASSWORD` （不建议设置）

`PGPASSFILE`  格式： `host:port:dbname:user:password`

如果`PGPASSFILE`没有被设置，会查询`~/.pgpass`文件

`PGPASSFILE`必须是`0600`权限 ，如果不是会被忽略不起作用

可以用`*`匹配所有情况，比如：`myhost:5432:*:sriggs:moresecurepw`

共享的`pg_service.conf`，默认在`/etc/pg_service.conf`，可以用`PGSYSCONFDIR`指定其他目录。 

独享的`~/.pg_service.conf`。配置格式：

```
[dbservice1]
host=postgres1
port=5432
dbname=postgres 
```

`PGSERVICEFILE`用于修改`pg_service.conf`这个名字

`PGSERVICE`环境变量也可以直接指定，配置`service`，如下方法使用

`service=dbservice1 user=sriggs`

`pg_service.conf`和`.pgpass`文件可以一起工作，一个用于共享目的简化，连接目标配置，一个主要用于配置密码。
 
`Uniform Resource Identifier` (URI) 格式: `postgresql://myuser:mypasswd@myhost:myport/mydb`

PGRELEASE

PGSERVERNAME

PGHOME

PGDATA
