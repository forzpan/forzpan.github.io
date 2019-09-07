---
title: PostgreSQL 之 SQL执行流程
date: 2019-09-06 01:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中SQL执行的各个步骤
---

* **建立连接**

PG的连接是一对一的，一个客户端进程对应一个服务端进程。服务端进程是由服务端主进程专门fork出来响应客户端进程的。

* **语法检查**

flex工具进行词法分析，把文件scan.l转换成C源文件scan.c，C编译器就可以用于创建词法分析器。

bison工具进行语法分析，把文件gram.y转换为gram.c，C编译器就可以用于创建语法分析器。

纯文本--->词法分析器--->语法分析器--->分析树--->转换处理--->查询树。

* **查询重写**

重写器的输入和输出都是查询树。

* **规划器/优化器**

参加join的表少会检查每一种可能的执行路径，当表比较多会使用geqo遗传查询优化器来选择执行计划。

先确定单表执行计划，再确定连接执行计划，最终生成计划树。

* **执行器**

执行器接手规划器/优化器创建的计划，并递归地处理之以抽取所需的行集。本质上是一种需求拉动的管道机制。

[PostgreSQL内部概述](http://www.postgres.cn/docs/11/overview.html)

[PostgreSQL SQL执行流程](http://blog.itpub.net/30088583/viewspace-1715489)

[PostgreSQL Select源码解析](http://blog.itpub.net/30088583/viewspace-1421547/)

