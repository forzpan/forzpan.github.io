---
title: PostgreSQL 之 常用管理SQL
date: 2019-12-05 04:00:00
categories:
- PostgreSQL-Administrate
tags:
- PostgreSQL
description: 介绍PostgreSQL中运维常用的一些SQL
---

```sql
SELECT current_database();

SELECT current_user;

SELECT inet_server_addr(), inet_server_port();

SELECT version();

SELECT current_time;

SELECT date_trunc('second', current_timestamp - pg_postmaster_start_time()) as uptime;

SELECT count(*) FROM information_schema.tables WHERE table_schema NOT IN ('information_schema', 'pg_catalog'); 

SELECT datname,pg_size_pretty(pg_database_size(datname)) size from pg_database;

select pg_relation_size('pgbench_accounts');
select pg_total_relation_size('pgbench_accounts');
\dt+   pgbench_accounts


SELECT table_schema||'.'||table_name as tbname,pg_relation_size(table_schema || '.' || table_name) as size
FROM information_schema.tables
WHERE table_schema NOT IN ('information_schema', 'pg_catalog')
ORDER BY size DESC
LIMIT 10;


SELECT (CASE WHEN reltuples > 0 THEN
pg_relation_size(oid)*reltuples/(8192*relpages)
ELSE 0
END)::bigint AS estimated_row_count
FROM pg_class
WHERE oid = 'mytable'::regclass;

pg_relation_size 会对表请求 AccessExclusiveLock ，所以，上面估算查询虽然代价低，但是可能会卡住。

所以办法是从操作系统文件系统去获取表文件大小。那么需要一些信息，表空间oid，数据库oid，表文件oid。

SELECT reltablespace, relfilenode FROM pg_class
WHERE oid = 'mytable'::regclass;

SELECT oid as databaseid FROM pg_database
WHERE datname = current_database();

如果reltablespace是0，那就是在base目录下面。
用函数实现：

CREATE OR REPLACE FUNCTION estimated_row_count(text)
RETURNS bigint
LANGUAGE sql
AS $$
SELECT (CASE WHEN reltuples > 0 THEN
pg_relation_size($1)*reltuples/(8192*relpages)
ELSE 0
END)::bigint
FROM pg_class
WHERE oid = $1::regclass;
$$;


CREATE OR REPLACE FUNCTION pg_relation_size_nolock(tablename regclass)
RETURNS BIGINT
LANGUAGE plpgsql
AS $$
DECLARE
classoutput RECORD;
tsid INTEGER;
rid INTEGER;
dbid INTEGER;
filepath TEXT;
filename TEXT;
datadir TEXT;
i INTEGER := 0;
tablesize BIGINT;
BEGIN
--
-- Get data directory
--
EXECUTE 'SHOW data_directory' INTO datadir;
--
-- Get relfilenode and reltablespace
--
SELECT
reltablespace as tsid
,relfilenode as rid
INTO classoutput
FROM pg_class
WHERE oid = tablename
AND relkind = 'r';
--
-- Throw an error if we can't find the tablename specified
--
IF NOT FOUND THEN
RAISE EXCEPTION 'tablename % not found', tablename;
END IF;
tsid := classoutput.tsid;
rid := classoutput.rid;
--
-- Get the database object identifier (oid)
--
SELECT oid INTO dbid
FROM pg_database
WHERE datname = current_database();
--
-- Use some internals knowledge to set the filepath
--
IF tsid = 0 THEN
filepath := datadir || '/base/' || dbid || '/' || rid;
ELSE
filepath := datadir || '/pg_tblspc/' || tsid || '/'
|| dbid || '/' || rid;
END IF;
--
-- Look for the first file. Report if missing
--
SELECT (pg_stat_file(filepath)).size
INTO tablesize;
--
-- Sum the sizes of additional files, if any
--
WHILE FOUND LOOP
i := i + 1;
filename := filepath || '.' || i;
--
-- pg_stat_file returns ERROR if it cannot see file
-- so we must trap the error and exit loop
--
BEGIN
SELECT tablesize + (pg_stat_file(filename)).size
INTO tablesize;
EXCEPTION
WHEN OTHERS THEN
EXIT;
END;
END LOOP;
RETURN tablesize;
END;
$$;


SELECT * FROM pg_extension;

SELECT * FROM pg_constraint WHERE confrelid = 'orders'::regclass;

SELECT name, setting, reset_val, source FROM pg_settings WHERE source = 'session';

set local work_mem = '16MB';  只能用在事务块中

--查询被修改过的配置，就是不是initdb之后的初始配置的结果
SELECT name, source, setting FROM pg_settings WHERE source != 'default' AND source != 'override' ORDER by 2, 1;       
```

