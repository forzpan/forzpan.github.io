---
title: PostgreSQL 之 常用管理SQL
date: 2019-12-05 04:00:00
categories:
- PostgreSQL-Administrate
tags:
- PostgreSQL
description: 介绍PostgreSQL中运维常用的一些SQL
---

# 系统信息函数

```sql
-- 查看版本信息
SELECT version();

-- 查看当前数据库名
SELECT current_database();
SELECT current_catalog;

-- 查看当前模式名
SELECT current_schema;

-- 查看搜索路径中的模式名，可以选择是否包含隐式模式
SELECT current_schemas(true);
SELECT current_schemas(false);

-- 搜索路径可以在运行时修改。
SET search_path TO schema [, schema, ...]

-- 查看会话用户
SELECT session_user;

-- 查看当前用户
SELECT user;
SELECT current_user;
SELECT current_role;

-- session_user通常是发起当前数据库连接的用户，不过超级用户可以用SET SESSION AUTHORIZATION修改这个设置。
-- current_user是用于权限检查的用户标识。通常，它总是等于会话用户，但是可以被SET ROLE改变。它也会在函数执行的过程中随着属性SECURITY DEFINER的改变而改变。

-- 查看本地连接的地址和端口
SELECT inet_client_addr(), inet_client_port();

-- 查看远程连接的地址和端口
SELECT inet_server_addr(), inet_server_port();

-- 一般使用是： pg_blocking_pids(pg_backend_pid())
-- 注意当一个预备事务持有一个冲突锁时，这个函数的结果中它将被表示为一个为0的进程ID。
-- 对这个函数的频繁调用可能对数据库性能有一些影响，因为它需要短时间地独占访问锁管理器的共享状态。

-- 查看当前时间相关
SELECT current_timestamp, current_date, current_time;

-- 服务器启动时间
SELECT pg_postmaster_start_time();

-- 服务器工作时长（秒）
SELECT date_trunc('second', current_timestamp - pg_postmaster_start_time()) as uptime;

-- 查看配置载入时间
SELECT pg_conf_load_time();

-- 当前日志收集器在使用的主日志文件名或者所要求格式的日志的文件名
SELECT pg_current_logfile([text]);

-- 会话的临时模式的 OID，如果没有则为 0
SELECT pg_my_temp_schema();

-- 模式是另一个会话的临时模式吗？
SELECT pg_is_other_temp_schema(oid);

-- 这个会话中JIT编译是否可用？如果jit被设置为off，则返回false。
SELECT pg_jit_available();

-- 会话当前正在监听的频道名称
SELECT pg_listening_channels();

-- 异步通知队列当前被占用的分数（0-1）
SELECT pg_notification_queue_usage();

-- 阻止指定服务器进程ID获取安全快照的进程ID
SELECT pg_safe_snapshot_blocking_pids(int);

-- PostgreSQL触发器的当前嵌套层次（如果没有调用则为 0，直接或间接，从一个触发器内部开始）
SELECT pg_trigger_depth();
```

# 数据库对象定位与系统文件操作

```sql
-- 指定关系的文件结点号
SELECT pg_relation_filenode(relation regclass);

-- 指定关系的文件路径名
SELECT pg_relation_filepath(relation regclass);

-- 查找与给定的表空间和文件节点相关的关系
SELECT pg_filenode_relation(tablespace oid, filenode oid);

-- 返回一个文本文件的内容。默认仅限于超级用户使用，但是可以给其他用户授予EXECUTE让他们运行这个函数。
SELECT pg_read_file(filename text [, offset bigint, length bigint [, missing_ok boolean] ])

-- 返回一个文件的内容。默认仅限于超级用户使用，但是可以给其他用户授予EXECUTE让他们运行这个函数。
SELECT pg_read_binary_file(filename text [, offset bigint, length bigint [, missing_ok boolean] ])	

-- 返回关于一个文件的信息。默认仅限于超级用户使用，但是可以给其他用户授予EXECUTE让他们运行这个函数。
SELECT pg_stat_file(filename text[, missing_ok boolean])
```

# 数据库存储尺寸相关

```sql

-- 把人类可读格式的带有单位的尺寸转换成字节数
select pg_size_bytes(text);
-- 将表示成一个 64位整数的字节尺寸转换为带尺寸单位的人类可读格式，基于1024而不是1000
select pg_size_pretty(bigint);

-- 存储一个特定值（可能压缩过）所需的字节数
select pg_column_size(any);

-- 指定数据库使用的磁盘空间
select pg_database_size(oid);
select pg_database_size(name);
-- 查看各个数据库的存储尺寸
SELECT datname,pg_size_pretty(pg_database_size(datname)) size from pg_database;

-- 指定的表空间使用的磁盘空间
select pg_tablespace_size(oid);
select pg_tablespace_size(name);

-- 附加到指定表的索引所占的总磁盘空间
select pg_indexes_size(regclass);
-- 被指定表使用的磁盘空间，排除索引（但包括TOAST、fsm和vm）
select pg_table_size(regclass);
-- 指定表所用的总磁盘空间，包括所有的索引和TOAST数据
-- 等价于pg_table_size + pg_indexes_size
select pg_total_relation_size(regclass);
-- 查看表存储情况
\dt+ pgbench_accounts

-- 指定表或索引的指定分叉（'main'、'fsm'、'vm'或'init'）使用的磁盘空间
-- 这是一个比较底层的函数，常规情况下使用上面几个高层函数即可
select pg_relation_size('pgbench_accounts', 'main');
-- pg_relation_size(..., 'main')的简写，
select pg_relation_size('pgbench_accounts');

-- 当前用户使用的表数量
SELECT count(*) FROM information_schema.tables WHERE table_schema NOT IN ('information_schema', 'pg_catalog'); 

-- 查看所有用户表main分叉的存储大小，按大小倒排
SELECT table_schema||'.'||table_name as tbname,pg_relation_size(table_schema || '.' || table_name) as size
FROM information_schema.tables
WHERE table_schema NOT IN ('information_schema', 'pg_catalog')
ORDER BY size DESC
LIMIT 10;

-- 估算表的总行数
SELECT (CASE WHEN reltuples > 0
THEN pg_relation_size(oid)*reltuples/(8192*relpages)
ELSE 0
END)::bigint AS estimated_row_count
FROM pg_class
WHERE oid = 'mytable'::regclass;

-- 估算表的总行数的函数实现：
CREATE OR REPLACE FUNCTION estimated_row_count(text)
RETURNS bigint
LANGUAGE sql
AS $$
    SELECT (CASE WHEN reltuples > 0
    THEN pg_relation_size($1)*reltuples/(8192*relpages)
    ELSE 0
    END)::bigint
    FROM pg_class
    WHERE oid = $1::regclass;
$$;

-- pg_relation_size 会对表请求 AccessExclusiveLock ，所以，上面估算查询虽然代价低，但是可能会卡住。
-- 所以办法是从操作系统文件系统去获取表文件大小。那么需要一些信息，表空间oid，数据库oid，表文件oid。

-- 获取表空间oid和表文件oid，如果reltablespace是0，那就是在base目录下面。
SELECT reltablespace, relfilenode
FROM pg_class
WHERE oid = 'mytable'::regclass;
-- 获取数据库oid
SELECT oid as databaseid
FROM pg_database
WHERE datname = current_database();

-- 获取表文件存储大小的函数实现
-- 函数或许有点问题，global目录怎么办，为什么不使用pg_relation_filepath获取路径
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
    -- Get reltablespace and relfilenode
    --
    SELECT reltablespace as tsid, relfilenode as rid
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
        filepath := datadir || '/pg_tblspc/' || tsid || '/' || dbid || '/' || rid;
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
```

# 数据库配置相关

```sql
-- 获得设置的当前值,missing_ok为true的话，即使不是合法setting_name，也不报错，返回null。
SELECT current_setting(setting_name [, missing_ok ]);
-- 当省略missing_ok，或者missing_ok为false时，效果等同于 show 命令。
show setting_name

-- 设置一个参数并返回新值，如果is_local设置为true，那么新值将只应用于当前事务。
SELECT set_config(setting_name, new_value, is_local)
-- 如果希望新值应用于当前会话，is_local应该使用false。它等效于 SQL 命令 SET。
set work_mem = '16MB';
-- 只能用在事务块中,临时调整工作内存
set local work_mem = '16MB';

-- 查看会话级别参数配置
SELECT name, setting, reset_val, source FROM pg_settings WHERE source = 'session';

--查询被修改过的配置，就是不是initdb之后的初始配置的结果
SELECT name, source, setting FROM pg_settings WHERE source != 'default' AND source != 'override' ORDER by 2, 1; 

-- 给服务器进程发一个SIGHUP信号，让其重载配置文件
SELECT pg_reload_conf();
```

# 服务器进程相关

```sql
-- 查看与当前会话关联的服务器进程的进程ID
SELECT pg_backend_pid();

-- 查看阻塞指定服务器进程ID获得锁的进程ID
SELECT pg_blocking_pids(int);

-- 给服务器进程发一个SIGINT信号，取消一个后端的当前查询。
-- 如果调用角色是被取消后端的拥有者角色的成员或者调用角色已经被授予pg_signal_backend。
-- 只有超级用户才能取消超级用户的后端。
SELECT pg_cancel_backend(pid int);

-- 给服务器进程发一个SIGTERM信号，中止一个后端。
-- 如果调用角色是被取消后端的拥有者角色的成员或者调用角色已经被授予pg_signal_backend。
-- 只有超级用户才能中止超级用户的后端。
SELECT pg_terminate_backend(pid int);

-- 一个活动后端的进程 ID可以从pg_stat_activity视图的pid列中找到，也可以从系统进程列表发现。

-- 给服务器进程发信号，让其切换服务器的日志文件。
SELECT pg_rotate_logfile();
```

# 其他

```sql
-- 查看插件
SELECT * FROM pg_extension;

-- 查看表约束
SELECT * FROM pg_constraint WHERE confrelid = 'orders'::regclass;

```

更多需访问 [PostgreSQL 系统信息函数](http://postgres.cn/docs/11/functions-info.html)
更多需访问 [PostgreSQL 系统管理函数](http://postgres.cn/docs/11/functions-admin.html)
