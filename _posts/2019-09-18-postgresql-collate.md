---
title: PostgreSQL 之 区域设置和排序规则
date: 2019-09-18 02:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: PostgreSQL中区域设置和排序规则相关内容
---

# 区域设置 

区域支持是在使用initdb创建一个数据库集簇时自动被初始化的。

`initdb --locale=en_US`

默认使用操作系统的区域设置，locale命令显示:

```
LANG=en_US.UTF-8
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
LC_COLLATE="en_US.UTF-8"
LC_MONETARY="en_US.UTF-8"
LC_MESSAGES="en_US.UTF-8"
LC_PAPER="en_US.UTF-8"
LC_NAME="en_US.UTF-8"
LC_ADDRESS="en_US.UTF-8"
LC_TELEPHONE="en_US.UTF-8"
LC_MEASUREMENT="en_US.UTF-8"
LC_IDENTIFICATION="en_US.UTF-8"
LC_ALL=
```

PostgreSQL中，支持以下配置，自定义本地规则，它们都是initdb的参数。

```
LC_COLLATE      字符串排序顺序
LC_CTYPE        字符分类（什么是一个字符？它的大写形式是否等效？）
LC_MESSAGES     消息使用的语言
LC_MONETARY     货币数量使用的格式
LC_NUMERIC      数字的格式
LC_TIME         日期和时间的格式
```

数据库一旦确定LC_COLLATE和LC_CTYPE后就不可修改，因为他们涉及到索引生成的规则。其他的几个可以随时修改，所以只有LC_MESSAGES、LC_MONETARY、LC_NUMERIC、LC_TIME在postgresql.conf中是可配置的。

由于LC_COLLATE和排序顺序相关，LC_CTYPE字符相关，能判断大小写是否等效。因此不同的配置，影响一下几个SQL特性：

* 在文本数据上使用ORDER BY或标准比较操作符的查询中的排序顺序。（'2'比'1'大好理解，几乎所有排序规则都是这么规定的，那么空格和换行符哪个大呢？可就不一定了。）
* 函数upper、lower和initcap。
* 模式匹配操作符（LIKE、SIMILAR TO和POSIX风格的正则表达式）；区域影响大小写不敏感匹配和通过字符类正则表达式的字符分类。
* to_char函数家族。
* 为LIKE子句使用索引的能力。

PostgreSQL中使用非C或非POSIX区域的缺点是`性能影响`。它`降低了字符处理的速度`并且`阻止了在LIKE中对普通索引的使用`。因此，`只能在真正需要的时候才使用它`。

# 排序规则

排序规则特性允许指定每一列甚至每一个操作的数据的排序顺序和字符分类行为。

* 一种可排序数据类型的每一种表达式都有一个排序规则（内建的可排序数据类型是text、varchar和char。用户定义的基础类型也可以被标记为可排序的，并且可排序数据类型上的域也是可排序的）。
* 如果该表达式是一个常量，它的应用排序规则就是该常量数据类型的默认排序规则（DataBase设置，或者域设置）。
* 如果该表达式是一个列引用，应用排序规则就是列所定义的排序规则（Column设置，Column未设置就使用域设置或者DataBase设置）。
* 自定义函数的排序规则是由函数的参数确定（显式或隐式）。
* 把操作符也当作函数来理解的话，它的排序规则也由参数确定（显式或隐式）。
* ORDER BY子句的排序规则是排序表达式的排序规则（显式或隐式）。

举个例子：

初始化，确定整个server级别的默认设置：

`initdb --locale=C -E UTF8`

创建数据库，确定数据库级别默认设置：

```sql
CREATE DATABASE testdb TEMPLATE template0 ENCODING UTF8 LC_COLLATE 'en_US.UTF-8' LC_CTYPE 'en_US.UTF-8';
```

创建域，确定域级别的默认设置：

```sql
CREATE DOMAIN nametxt AS TEXT COLLATE 'fr_FR';
```

创建表，确定字段级别的默认设置：

```sql
create table testtb
(
    enname text,                        --使用数据库默认设置
    frname nametxt,                     --使用域设置
    dename text COLLATE "de_DE",        --使用列设置
    chname text COLLATE "zh_CN"
);
```

除ORDER BY子句、自定义函数、比较操作符之外，在大小写字母之间转换的函数会考虑排序规则，例如lower、upper和initcap。模式匹配操作符和to_char及相关函数也会考虑排序规则。

```sql
SELECT enname < 'foo' FROM testtb;                                  --使用库排序规则
SELECT frname < 'foo' FROM testtb;                                  --使用域排序规则
SELECT dename < 'foo' FROM testtb;                                  --使用列排序规则
 
SELECT enname < 'foo' COLLATE "fr_FR" FROM testtb;                  --使用显式指定的排序规则
SELECT dename < chname COLLATE "fr_FR" FROM testtb;                 --使用显式指定的排序规则
SELECT dename COLLATE "fr_FR" < chname FROM testtb;                 --同上，是等效的
 
SELECT enname < frname FROM testtb;                                 --使用frname的域排序规则
SELECT enname < chname FROM testtb;                                 --使用chname的列排序规则
 
SELECT dename < chname FROM testtb;                                 --两个不同的列排序规则，冲突报错
SELECT frname < dename FROM testtb;                                 --不同的域排序规则和列排序规则，冲突报错
SELECT dename COLLATE "fr_FR" < chname COLLATE "es_ES" FROM testtb; --不同的显式排序规则
```

连接操作符好虽然结构类似，但是并不需要排序规则：

```sql
SELECT dename || chname FROM testtb;
```

但并不意味着它在ORDER BY子句中也能正常运行：

```sql
SELECT * FROM testtb order by dename || chname;                     --报错
```

因为连接操作虽然不需要排序规则，但是它的操作结果是一种可排序类型，且外围表达式（order by子句）需要排序规则，所以处理时，会使用排序规则。

```sql
SELECT * FROM testtb order by dename || chname COLLATE "fr_FR";
```

总结，当多个排序规则需要被组合时，将使用下面的规则：

* 如果任何一个输入表达式具有一个显式排序规则派生，则在输入表达式之间的所有显式派生的排序规则必须相同，否则将产生一个错误。如果任何一个显式派生的排序规则存在，它就是排序规则组合的结果。
* 否则，所有输入表达式必须具有相同的隐式排序规则派生或默认排序规则。如果任何一个非默认排序规则存在，它就是排序规则组合的结果。否则，结果是默认排序规则。
* 如果在输入表达式之间存在冲突的非默认隐式排序规则，则组合被认为是具有不确定排序规则。这并非一种错误情况，除非被调用的特定函数要求提供排序规则的知识。如果它确实这样做，运行时将发生一个错误。

# 索引使用 

一个索引在每一个索引列上只能支持一种排序规则。因此查询使用的排序规则和索引的排序规则不同时，不能够使用该索引。可以创建多个索引支持不同的排序规则。

```sql
CREATE TABLE test1c (
    id integer,
    content varchar COLLATE "x"
);
CREATE INDEX test1c_content_index ON test1c (content);
```

索引 test1c_content_index 使用了排序规则 "x"：

```sql
SELECT * FROM test1c WHERE content > constant;              --可以使用索引test1c_content_index
SELECT * FROM test1c WHERE content > constant COLLATE "y";  --不能使用索引test1c_content_index
```

需要再创建一个"y"排序规则的索引去适用查询：

```sql
CREATE INDEX test1c_content_y_index ON test1c (content COLLATE "y");
```



