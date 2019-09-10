---
title: PostgreSQL 之 物理存储
date: 2019-09-08 03:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中物理存储相关内容
---

# 目录与文件

对于集簇里的每个数据库，在表空间目录(base或者自定义)里都有一个子目录对应，子目录的名字为该数据库在pg_database里的OID。

每个表和索引都存储在独立的文件里。对于普通关系，这些文件以表或索引的filenode命名，它可以在pg_class.relfilenode中找到。(这个filenode是pg自己内部维护的，我第一次竟然理解成了文件系统的，尴尬)。

注意：

虽然普通表的filenode通常和它的pg_class.OID相匹配，但实际上并不必须如此。

有些操作，比如TRUNCATE、REINDEX、CLUSTER以及某些形式的ALTER TABLE，都可以改变pg_class.relfilenode而同时保留pg_class.OID。（试了一下truncate，确实改变了，还发现了显式事务内部truncate能回滚的原理）

此外，对于特定系统目录，其pg_class.relfilenode包含0。这些目录的实际文件节点号被存储在一个低层数据结构pg_filenode.map中，并且可以使用pg_relation_filenode()函数获取。

[PostgreSQL 物理文件映射解析](http://blog.itpub.net/30088583/viewspace-1633683/)

每个表和索引有一个空闲空间映射(FSM)文件，该文件以filenode加上后缀_fsm命名，它存储可用空闲空间的信息。

每个表还有一个可见性映射(VM)，存储在一个后缀为_vm的文件中，它用于跟踪哪些页面已知含有非死亡元组。

不被日志记录的表和索引还有一个_init文件，即初始化文件，崩溃时，用于复制恢复，其他_fsm和_vm之类的文件会被直接擦除。

对于临时表，文件名的形式为tBBB_FFF，其中BBB是创建该文件的backend id，FFF是filenode。

表或者索引段超过segsize(编译时可配置)之后，就会被切分，后续段被命名为 filenode.1、filenode.2等等。原则上，_fsm和_vm文件也可以有多个段，但实际上很少发生。

表列中存储相当大的项，那么该表就会有个与之相关联的TOAST表，用于存储无法保留在在表行中的字段值。如果表有TOAST表，该表的pg_class.reltoastrelid链接到它的TOAST表。(TOAST部分详细介绍)。

用户自定义表空间在pg_tblspc目录里面有一个用该表空间的OID命名的符号连接，它指向物理的表空间目录。

临时文件(例如排序内存不足时产生的)被创建在 临时表空间下的pgsql_tmp目录中，临时文件的名称的形式为pgsql_tmpPPP.NNN，其中PPP是其所属后端的PID，而NNN用于区别该后端的不同临时文件。

pg_relation_filepath()函数显示任何关系的完整路径(相对于PGDATA)。

# 空闲空间映射(FSM)

每一个堆和索引关系(除了哈希索引)都有一个空闲空间映射(FSM)来保持对关系中可用空间的跟踪。

[PostgreSQL 之 FSM](https://forzpan.github.io/postgresql-kernel/2019/09/08/postgresql-fsm/)

# 可见性映射 (VM)

_VM文件为每个堆页面存储两个bit。

第一bit被设置，表示该页面上的元组都是有效的。index-only scans会利用这个bit。

第二bit被设置，表示该页面上的元组都已经被冻结。防回卷清理操作也不需要重新访问该页面。

可见性映射的位只会被清理操作设置，但是可以被任何在页面上进行的数据修改操作清除。

# 数据库页面布局

术语item指的是存储在一个页面里的 独立数据值。在一个表里，一个item是一个行。在一个索引里，一个item是一条索引记录。序列和TOAST表的格式就像一个普通表一样。

页面总体布局(所有细节都可以在src/include/storage/bufpage.h中找到)。

```
PageHeaderData  24字节长。包含关于页面的一般信息，包括空闲空间指针。
ItemIdData      存放(偏移量，长度)对的数组，指向实际Item。每个4字节。一个Item会因为压缩空闲空间在页面内进行移动，但对应的ItemIdData不会移动。实际上，每个指向Item的指针(ItemPointer，也叫做CTID)都由一个blocknumber和一个ItemIdData的索引组成。
Free space      未分配的空间(空闲空间)。新Item指针从这个区域的开头开始分配，新Item从其结尾开始分配。
Items           实际的Item本身。
Special space   索引访问模式相关的数据。不同的索引访问方式存放不同的数据。比如，b-tree 索引用它存储指向页面的左右兄妹的链接，以及其他一些和索引结构相关的数据。普通表并不使用这个部分(通过设置pd_special等于页面大小来做到)。
```

PageHeaderData布局:

```
pd_lsn              PageXLogRecPtr  8 bytes     LSN: 最后修改这个页面的 WAL 记录最后一个字节后面的第一个字节
pd_checksum         uint16          2 bytes     页面校验码（当data checksums被设置，这个域是有效的，否则可以忽略）
pd_flags            uint16          2 bytes     标志位，标识页面存储情况
pd_lower            LocationIndex   2 bytes     到空闲空间开头的偏移量
pd_upper            LocationIndex   2 bytes     到空闲空间结尾的偏移量
pd_special          LocationIndex   2 bytes     到特殊空间开头的偏移量
pd_pagesize_version uint16          2 bytes     页面大小和布局版本号信息
pd_prune_xid        TransactionId   4 bytes     页面上最老未删除XMAX，如果没有则为0
```

HeapTupleHeaderData布局(表和序列都使用的Item结构的行头，所有细节都可以在src/include/access/htup_details.h中找到)。

```
t_xmin              TransactionId   4 bytes     插入XID标志
t_xmax              TransactionId   4 bytes     删除XID标志
t_cid               CommandId       4 bytes     插入和/或删除CID标志(覆盖t_xvac)
t_xvac              TransactionId   4 bytes     VACUUM操作移动一个行版本的XID
t_ctid              ItemPointerData 6 bytes     当前版本的CTID或者指向更新的行版本
t_infomask2         uint16          2 bytes     一些属性，加上多个标志位
t_infomask          uint16          2 bytes     多个标志位
t_hoff              uint8           1 byte      到用户数据的偏移量
```

数据的存储:

```
HeapTupleHeaderData
空值位图（可能没有）
对齐填充
OID（可能没有）
其他字段，在行内的具体位置，依靠pg_attribute中，attlen和attalign确定
```

[PostgreSQL中page页结构](http://blog.itpub.net/30088583/viewspace-1387177/)

# TOAST(超尺寸属性存储技术)

PostgreSQL使用固定的页面尺寸(通常是8kB)，并且不允许元组跨越多个页面。因此不可能直接存储非常大的字段值。为了克服这个限制，大的字段值会被压缩并/或分解成多个物理行。

只有变长(varlena)数据类型支持TOAST，通常在存储值的头四个字节，表示值的总字节数。

TOAST占用变长类型的长度字的最高两个2个bit，也就是剩下的30位可以用来表示大小，1GB。

```
1XXXXXXX                               变长类型值简短存储。剩余7bit表示(正常值+长度字)有多少字节。适合127字节以内的值。(大多数变长数据)
00XXXXXX XXXXXXXX XXXXXXXX XXXXXXXX    变长类型值普通存储。剩余30bit表示(正常值+长度字)有多少字节。
01XXXXXX XXXXXXXX XXXXXXXX XXXXXXXX    变长类型值压缩存储。剩余30bit表示(压缩值+长度字)有多少字节。
10000000 XXXXXXXX                      变长类型值线外存储。表示一个TOAST指针，真正的值在TOAST表里。第二个字节决定这个TOAST指针的类型和尺寸。
```

如果表包含main,extended或external存储策略的字段时，那么该表将有一个关联的TOAST表(如果可以计算出来不需要TOAST表则不会有，比如两个varchar(10)字段这种)。其OID存储在表的pg_class.reltoastrelid项中。

被TOAST过的值保存在TOAST表里。值或压缩过的值被分裂成最大为TOAST_MAX_CHUNK_SIZE(通常是blocksize的1/4)字节的块。每个块都作为独立的行存储在TOAST表中。

TOAST表有三个字段：chunk_id(被TOAST值的OID)，chunk_seq(块序列号，表示在值中第几个块)和一个chunk_data(块实际数据)。在(chunk_id,chunk_seq)上有唯一索引，进行快速检索。

TOAST后原位置存储共18byte：10000000 XXXXXXXX(2byte) + 值原始尺寸(4byte) + 物理存储尺寸(4byte) + chunk_id(4byte) + TOAST表OID(4byte)。

TOAST指针结构：

```
struct varatt_external 
{ 
    int32   va_rawsize;        /* Original data size (includes header) */ 
    int32   va_extsize;        /* External saved size (doesn't) */ 
    Oid     va_valueid;        /* Unique ID of value within TOAST table */ 
    Oid     va_toastrelid;    /* RelID of TOAST table containing it */ 
};
```

只有在准备向表中存储超过TOAST_TUPLE_THRESHOLD(通常是blocksize的1/4)字节的行值的时候才会触发TOAST管理代码toast_insert_or_update()，压缩或线外存储字段值，

直到行值比TOAST_TUPLE_TARGET(通常也是blocksize的1/4)字节短，或者无法得到更好的结果的时候才停止。

update时没有更新的线外TOAST值保持不动。

TOAST管理代码识别四种在磁盘上存储可TOAST列的策略：

```
PLAIN       避免压缩或者线外存储,而且对于变长类型禁用单字节头部。这是不可TOAST数据类型列的唯一策略，非变长类型当然也可以用。
EXTENDED    允许压缩和线外存储。这是大多数可TOAST数据类型的默认策略。先尝试压缩，如果行仍然太大，则进行线外存储。
EXTERNAL    允许线外存储，但不许压缩。将让宽text和bytea列上的操作更快(空间换时间)。
MAIN        允许压缩，但不允许线外存储(实际上，仍然会进行线外存储，但只作为没办法把行变得放入一页的最后手段)。
```

行存储逻辑中变长类型字段存储方式：

```
（1）计算TUPLESIZE(提一下127字节及以内的PLAIN策略变长字段是禁用单字节头部的，这是存储时的事情，和计算TUPLESIZE关系不大)。
（2）TUPLESIZE小于TOAST_TUPLE_THRESHOLD情况下，普通存储。
（3）TUPLESIZE大于TOAST_TUPLE_THRESHOLD，会触发TOAST管理代码，EXTENDED策略的变长字段按顺序依次压缩并重计算TUPLESIZE判断是否小于TOAST_TUPLE_TARGET，一旦小于立刻停止，不会继续压缩后续字段。
（4）EXTENDED策略的变长字段全部压缩后，TUPLESIZE仍然小于TOAST_TUPLE_TARGET，则需要进行线外存储。EXTENDED和EXTERNAL策略的变长字段按顺序依次将压缩后值(EXTENDED)或未压缩值(EXTERNAL)分裂成最大TOAST_MAX_CHUNK_SIZE大小的N个块存储到TOAST表中，并重计算TUPLESIZE判断是否小于TOAST_TUPLE_TARGET，一旦小于立刻停止，不会继续处理后续的变长类型字段。
（5）EXTENDED和EXTERNAL策略的变长字段全部线外存储后，TUPLESIZE仍然大于TOAST_TUPLE_TARGET。MAIN策略的变长字段按顺序依次压缩并重计算TUPLESIZE判断是否小于TOAST_TUPLE_TARGET，一旦小于立刻停止，不会继续压缩后续字段。
（6）MAIN策略的变长字段全部压缩后，TUPLESIZE仍然大于TOAST_TUPLE_TARGET。MAIN策略的变长字段按顺序依次进行线外存储并重计算TUPLESIZE判断是否小于TOAST_TUPLE_TARGET，一旦小于立刻停止，不会继续处理后续字段。
```

[TOAST介绍](https://github.com/digoal/blog/blob/master/201103/20110329_01.md)

[数据库物理存储](http://www.postgres.cn/docs/11/storage.html)







