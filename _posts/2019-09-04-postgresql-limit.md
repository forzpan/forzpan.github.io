---
title: PostgreSQL一些限制
date: 2019-09-04 06:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: 使用PostgreSQL需要了解的一些限制
---

表名、列名、函数名最大长度为63个字符,可以在编译前修改,修改后必须重新initdb

`src/include/pg_config_manual.h`中

```c
#define NAMEDATALEN 64
```

* 最大数据库大小无限制 
* 单个表最大大小 32 TB （对于block_size 8k，深层次原因是每个block都有个blocknumber，而blocknumber是个uint32类型）
* 最大行大小 1.6 TB 
* 最大字段大小 1 GB 
* 每张表的最大行数无限制 
* 每个表的最大列数 250 - 1600 取决于列类型 
* 每张表的最大索引数无限制


