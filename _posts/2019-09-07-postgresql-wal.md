---
title: PostgreSQL 之 WAL
date: 2019-09-07 02:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中WAL相关内容
---

WAL在pg_wal目录里，每个文件通常16MB，因为每个wal段是16M，由--with-wal-segsize=SEGSIZE控制。调整这个值来控制传送 WAL 日志的粒度非常有用。

每个段分割成多个页，通常每个页为8K，由--with-wal-blocksize控制，文件名从000000010000000000000000开始。

WAL的目的是确保在数据库记录被修改之前先写了日志。但是由于操作系统原因，write实际可能数据只是写到缓冲区，就返回写成功，这个时候突然断电，可导致，数据并未落盘。这点dba应该注意。

在完成一个检查点并且刷写了日志文件之后，检查点的位置被保存在文件pg_control里。因此在恢复的开始， 服务器首先读取pg_control，然后读取检查点记录； 接着它通过从检查点记录里标识的日志位置开始向后扫描执行 REDO操作。

当每个新记录被写入时，WAL记录被追加到WAL日志中。 插入位置由日志序列号（LSN）描述，该日志序列号是日志中的字节偏移量，类似1B34D846/B374D848。

 

扯一下，数据落盘：

首先，有操作系统的高速缓存。

然后，在磁盘驱动器的控制器上可能还有一个高速缓存。这在RAID控制卡上是特别常见的。有些高速缓存是直写式的，即写入动作在到达的时候就立刻写入到磁盘上。其它是回写式的，即发送给驱动器的数据在稍后的某个时间写入驱动器。

最后，大多数磁盘驱动器都有高速缓存。有些是直写的，有些是回写的， 和磁盘控制器一样，回写的磁盘高速缓存也存在数据丢失的问题。

在Linux上，可以使用hdparm -I查询IDE和SATA驱动器，如果在Write cache之后有一个*则表示写高速缓存被启用。可以用hdparm -W 0来关闭写高速缓存。

贴几篇文章:
    * [漫谈linux文件IO--io流程讲的很清楚](https://www.cnblogs.com/losing-1216/p/5073051.html)
    * [Linux文件系统(五)---三大缓冲区之buffer块缓冲区](https://blog.csdn.net/shanshanpt/article/details/39258373)
    * [再探Linux内核write系统调用操作的原子性](https://blog.csdn.net/dog250/article/details/78879600)

磁盘盘片会被分割为扇区，通常每个扇区512字节。每次物理读写都对整个扇区进行操作。实际现在的盘512是为了向下兼容，真实写都是4K了。pg默认块大小是8K，或者更大，也就是写的过程中断电可能导致部分扇区写了数据，部分扇区没成功的时候。

为了避免这样的失效，full_page_writes 参数打开，PostgreSQL在一个检查点之后第一次修改磁盘上的实际页面之前， 整个页面（这里只pg的页大小，默认8K）的映像写入永久WAL存储。就可以通过检查点之后的页面镜像和后续wal日志，恢复这个页的数据，这也理解到了wal落盘的重要性了。

 

WAL文件中的每一个记录都被一个CRC-32（32位）校验码所保护，这让我们可以判断记录内容是否正确。CRC值在我们写入每一个WAL记录时设置，并且在崩溃恢复、归档恢复和复制时检查。循环冗余校验码，没有纠错能力。

硬盘这东西，容易受到外界强电磁环境影响，太阳风能影响，但是一般吹不大大气层，打无人机的电磁枪，不知道对着来一下，影响大不大。

内存电压都比较高，所以比较稳定，但不排除内存某些颗粒坏了，操作系统应该能发现，避开这些page吧。

 

目前数据页并没有默认地被校验，但是WAL记录中记录的整页映像将被保护。可以使用 initdb --data-checksums 参数，初始化数据库集簇的时候开启。在数据页面上使用校验码来帮助检测 I/O 系统造成的损坏。启用校验码将会引起显著的性能惩罚。这个选项只能在初始化期间被设置，并且以后不能修改。如果被设置，在所有数据库中会为所有对象计算校验码。毕竟某些情况下磁盘上的一个bit，由1变成0还是有可能的。

pg_twophase中的单个状态文件被CRC-32保护。

 

用在大型SQL查询中排序的临时数据库文件、物化和中间结果目前没有被校验，对于这些文件的改变也不会导致写入WAL记录。

 

异步提交，默认情况下，pg是同步提交的synchronous_commit 是on。同步提交的意思是commit命令返回成功之后，保证相关的wal落盘。小事务过多时候，fsync可能调用过多，IO利用不好。允许异步提交的话，用以根据一些机制，刷wal，提高IO利用效率。

drop table，两阶段提交命令等无视异步提交设置，使用同步提交。

 

wal_level 日志级别

* logical会在replica基础上增加支持逻辑解码所需的信息。
* replica，它写入足够的数据以支持WAL归档和复制。默认值。
* minimal删除除了从崩溃或立即关闭中恢复所需的信息之外的所有日志记录。某些批量操作的WAL日志可以被安全地跳过，这可以使一些操作更快：[CREATE TABLE AS | CREATE INDEX | CLUSTER | COPY到在同一个事务中被创建或截断的表中]

fsync 默认on，pg将尝试确保更新被物理地写入到磁盘，做法是每次遇到 写命令 后发出fsync()系统调用，保证操作系统写缓存里的数据落盘。

full_page_writes 默认on，pg服务器在一个检查点之后的页面的第一次修改期间将每个页面的全部内容写到 WAL中。并且在关闭fsync的情况下也应该关闭它。

wal_log_hints 默认off，为on时，pg服务器一个检查点之后页面被第一次修改期间把该磁盘页面的整个内容都写入WAL，即使对所谓的提示位做非关键修改也会这样做。如果启用了数据校验和，提示位更新总是会被WAL记录并且这个设置会被忽略。

wal_compression 当这个参数为on，full_page_writes为on或者基础备份中，pg会压缩page镜像。

wal_sync_method 指定WAL flush到磁盘的方法。如果fsync是off，那么这个设置就可以忽略，因为WAL文件落盘将不会被强制。事务commit的时候才用到。

* open_datasync（用open()选项O_DSYNC写 WAL 文件）
* fdatasync（在每次提交时调用fdatasync()）
* fsync（在每次提交时调用fsync()）
* fsync_writethrough（在每次提交时调用fsync()，强制任何磁盘写高速缓存的直通写）
* open_sync（用open()选项O_SYNC写 WAL 文件）

synchronous_commit 事务同步提交，默认on，一个事务是否需要等待 WAL 记录被写操作完成，如果fsync是off，写的操作系统写缓存就返回。当设置为off时，在向客户端报告成功和真正保证事务不会被服务器崩溃威胁之间会有延迟（最大的延迟是wal_writer_delay的三倍）。如果synchronous_standby_names被设置：

* on:备机日志落盘
* remote_apply:备机日志应用，备机读可见
* remote_write:备机日志写到操作系统缓存，备机pg实例崩溃不影响
* local:本机日志落盘
* off:异步提交，落盘的最大的延迟是wal_writer_delay的三倍。

wal_buffers 用于还未写入磁盘的 WAL 数据的共享内存量。默认值 -1 选择等于shared_buffers的1/32的尺寸，但是不小于64kB也不大于WAL段的尺寸（通常为16MB）。手动指定32k以下圆整成32k。在每次事务提交时，WAL缓冲区的内容被写出到磁盘，因此极大的值不可能提供显著的收益。重度更新事务会产生大量wal，事务过程中，下面两个参数控制利用wal_buffer。

wal_writer_delay 默认值是200ms。 上次wal writer在刷新WAL之后，它会睡眠wal_writer_delay毫秒，除非被异步提交的事务唤醒。

wal_writer_flush_after 默认是1MB。 XLogInsertRecord写wal写到wal buffer，wal buffer不足时，会刷出部分wal到操作系统写缓存，刷出部分达到1M时，启动wal writer。

commit_delay 在一次WAL物理同步被发起之前，增加一个commit_delay时间延迟，以微秒计，精度为10ms的机器上，1-10000us一个样。fsync为on的场合才有效，在异步提交时被忽略。默认的commit_delay是零（无延迟）。

commit_siblings 在执行commit_delay延迟时，要求的并发活动事务的最小数目。默认值是五个事务。

上面2个参数 当有commit_siblings个并发的活动事务时，wal同步延时有个commit_delay的短暂等待，可以合并多个事务提交的wal，也就是合并这些wal在一次IO中。

commit_delay 定义了一个组提交领导者进程在XLogFlush中要求一个锁之后将会休眠的微秒数，而组提交追随者都排队等候在领导者之后。这样的延迟可以允许其它服务器进程把它们提交的记录追加到WAL buffer中，这样所有的这些记录将会被领导者的最终同步操作刷出。由于commit_delay的目的是允许每次刷写操作的开销能够在并发提交的事务之间进行分摊（可能会以事务延迟为代价，可能是10ms）。

pg_test_fsync程序可以被用来衡量一次WAL刷写操作需要的平均微秒数。该程序报告的一次8kB写操作后的刷出所用的平均时间的一半常常是commit_delay最有效的设置。sata和sas盘调整这个比较有效。


checkpoint_timeout 自动 WAL 检查点之间的最长时间，以秒计。默认是 5 分钟（5min）。

max_wal_size 在自动WAL检查点使得WAL增长到最大尺寸，达到这个尺寸，也会刷盘。默认1G，这是软限制。

当距离上个检查点时间到达checkpoint_timeout，或者这个时间内，wal产生了max_wal_size大小，执行一个新的检查点。没有新wal，就跳过本次检查点。

checkpoint_warning 如果由于填充检查点段文件导致的检查点之间的间隔低于这个参数表示的秒数，那么就向服务器日志写一个消息（建议增加max_wal_size的值）。默认值30s。零则关闭警告。

checkpoint_completion_target 指定检查点完成的目标，作为检查点之间总时间的一部分。默认是 0.5，IO尽量均匀分布在前50%的时间。检查点之间的时间是动态的，具体IO分布时间，应该是基于上一次检查点之间的时间来算的。

checkpoint_flush_after 在执行检查点时，只要有checkpoint_flush_after字节被写入操作系统写缓存，就尝试强制 OS 把这些写发送到底层存储。

min_wal_size wal文件小于这个值时，不会被删除，而是回收。就是空间有pg保持用以重用，不释放给操作系统。


检查点是在事务序列中的点，这种点保证被更新的堆和索引数据文件的所有信息在该检查点之前已被写入磁盘。

在完成一个检查点并且刷写了日志文件之后，检查点的位置被保存在文件pg_control里。

当PostgreSQL触发checkpoint发生的条件后，会调用CreateCheckPoint函数创建具体的检查点，具体过程如下：

* 遍历所有的数据buffer，将脏页块状态从BM_DIRTY改为BM_CHECKPOINT_NEEDED，表示这些脏页将要被checkpoint刷新到磁盘
* 调用CheckPointGuts函数将共享内存中的脏页刷出到磁盘
* 生成新的Checkpoint 记录写入到XLOG中
* 更新控制文件、共享内存里XlogCtl的检查点相关成员、检查点的统计信息结构

XLogInsertRecord用于向shared buffer中的WAL buffer里写一个新记录。XLogInsertRecord用于每次数据库低层修改（比如，记录插入）时都要在受影响的数据页上持有一个排它锁，因此该操作需要越快越好。但是如果没有空间存放新记录，那么XLogInsertRecord就向内核缓存里写出一段WAL buffer里的wal。写出WAL buffer可能还会强制创建新的wal日志段，消耗更多时间。

正常情况下，WAL buffer应该由一个XLogFlush请求来写和刷出，在大部分时候它都是发生在事务提交的时候以确保事务记录被刷写到永久存储。wal_writer_delay和wal_writer_flush_after，应该也能够控制这个XLogFlush请求。


总结wal落盘：

* 每隔wal_writer_delay时间，wal_writer会刷出wal buffer里的内容一次。
* 事务过程中，XLogInsertRecord写wal写到wal buffer，wal buffer不足时，会刷出部分wal到操作系统写缓存，刷出达到wal_writer_flush_after字节会调用fsync()一次。
* commit时，XLogFlush刷出（该事务的还是全部的？）wal buffer中的wal到操作系统写缓存，并用wal_sync_method指定的同步方法落盘。
* checkpoint时，每个脏页刷出内存之前，脏页头的记录的LSN之前的wal都要落盘。
 


[PgSQL · 特性分析 · Write-Ahead Logging机制浅析](http://mysql.taobao.org/monthly/2017/03/02)

[PgSQL · 特性分析 · checkpoint机制浅析](http://mysql.taobao.org/monthly/2017/04/04)

[PostgreSQL持久性优化机制——WAL](https://www.jianshu.com/p/a37ceed648a8)



