---
title: PostgreSQL 之 FSM
date: 2019-09-08 02:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中 Free Space Map 相关内容
---

# 介绍 

由于PostgreSQL的MVCC机制，过期版本的元组（tuple）所占的物理空间被vacuum释放后需要尽可能重新利用。

因此设计了FSM（Free Space Map），为每一个堆和索引关系（哈希索引除外）都维护一个FSM文件来保持对关系中可用空间的跟踪，用于快速找到一个有足够的空闲空间用来容纳新增的数据的页，如果没有这样的页，则分配新页。

FSM文件在第一次vacuum时创建，初始大小3个PAGE（一般为24KB），后续每次vacuum时更新。

备注：

* 对于索引，跟踪的是完全未使用的页面，而不是页面中的可用空间。
* 最新代码里为了节省空间，小于 HEAP_FSM_CREATION_THRESHOLD （默认是4）个page的表也不会有FSM，因为额外3个PAGE太浪费了 （哪个版本发布不知道呢）。

# 原理

## 档位

FSM的空间管理是粗粒度的，每个heap page用一个map byte表示，共可表示2的8次方即256个档位。比如对于一个8K的page，map byte用0表示有0-31byte的空闲空间，1表示有32-63byte的空闲空间。

换个角度，0不能保证有空间可用，1保证能分配32byte，2保证能分配64byte，以此类推。可以看出来存在浪费，但是加速了空间的分配效率，是一个用空间换时间的设计。

8K的页，设想如下：

```
    空闲空间        档位
   ---------------------
    0    - 31       0
    32   - 63       1
    ...    ...      ...
    8128 - 8159     254
    8160 - 8191     255
```

事实如下：

```
    空闲空间        档位
   ---------------------
    0    - 31       0
    32   - 63       1
    ...    ...      ...
    8128 - 8163     254
    8164 - 8192     255
```

可以看出最后两个档位和想象的不一致，因为对于一个page来说，至少要存一个tuple吧。

因此，最大请求空间 MaxFSMRequestSize 就是最大Tuple长度 MaxHeapTupleSize，就是8K减去页头和一个tuple指针的大小（页结构看src/include/storage/bufpage.h），也就是8192-24-4=8164。

```c
#define MaxFSMRequestSize   MaxHeapTupleSize
#define MaxHeapTupleSize    (BLCKSZ - MAXALIGN(SizeOfPageHeaderData + sizeof(ItemIdData)))
```

如果采用设想方案，申请MaxFSMRequestSize，也就是8164byte的大小的空间是没法从FSM快速分配的，因为255档位只能保证有8160大小的空间可用。

因此，对最后两个档位进行调整，让MaxFSMRequestSize成为255档位的下边界，得到事实方案，可以保证255档位满足MaxFSMRequestSize可用空间的需求。

代码中请求的字节数和档位之间的转换函数如下：

```c
#define FSM_CATEGORIES  256
#define FSM_CAT_STEP    (BLCKSZ / FSM_CATEGORIES)
 
static uint8
fsm_space_needed_to_cat(Size needed)
{
    int         cat;
 
    /* Can't ask for more space than the highest category represents */
    if (needed > MaxFSMRequestSize)
        elog(ERROR, "invalid FSM request size %zu", needed);
 
    if (needed == 0)
        return 1;
 
    cat = (needed + FSM_CAT_STEP - 1) / FSM_CAT_STEP;
 
    if (cat > 255)
        cat = 255;
 
    return (uint8) cat;
}
```

## 二叉树

粗粒度用空间换时间获得了性能，但是还不够，想象一下一个FSM页中map byte像数组一样存储，每次找符合要求的页就是顺序遍历，这是个很low的方案。

于是有了二叉树方案，每两个map byte一组，再分配一个byte作为上层记录这两个byte中较大的值，重复这个操作，完成一个[完全二叉树](https://www.cnblogs.com/myjavascript/articles/4092746.html)。

![](/images/201909/20.png)

这样从根节点开始一路往下就能很快找到符合要求的heap页了，如果根节点不满足，直接分配新的page，这大大减少了判断次数。至于找到多个符合要求的路径怎么办？后面源码分析中有答案。

学习C那会儿形成一个印象，树状结构都需要指针，后来了解存储引擎持久化相关的技术才知道，树状的东西放到一个大的数组中，才方便持久化到硬盘，指针变成数组的index来存储。

FSM页也是需要持久化到磁盘的，由于完全二叉树的巧妙，可以直接计算子节点所在index，不需要额外的存储来记录。

![](/images/201909/21.png)

如上图所示：根节点在byte数组中的位置是0，两个子节点分别是1和2，1的子节点分别是3和4，可以得到n位置的子节点数组位置是2n+1和2n+2，父节点位置是(n-1)/2。

可以看出总节点数大概是\\(2^0+2^1+...+2^n\\)，其中最后一项是叶子节数目即最初的map byte们，大约占总存储空间的一半，也就是说树状结构额外增加了一倍的空间，来加速寻找符合条件的页，也是一种空间换时间的设计。

## High-level

首先介绍一下FSM页结构：

```
PageHeaderData      页头                    24byte
FSMPageData         FSM页数据
```

而FSMPageData结构体是：

```
fp_next_slot        下次搜索的开始位置       8byte
fp_nodes            存储树装数据的byte数组   变长
```

可以看出FSM页中可用于存储树状数据的fp_nodes空间为：

```c
#define NodesPerPage (BLCKSZ - MAXALIGN(SizeOfPageHeaderData) - offsetof(FSMPageData, fp_nodes))
```

其中SizeOfPageHeaderData就是页头大小，offsetof(FSMPageData, fp_nodes)就是fp_next_slot大小，至于fp_next_slot是有什么作用，后面源码分析总结。

代码中还规定了非叶子节点和叶子节点的可用空间为：

```c
#define NonLeafNodesPerPage (BLCKSZ / 2 - 1)
#define LeafNodesPerPage    (NodesPerPage - NonLeafNodesPerPage)
```

将一个FSM页看成一个黑盒，它能够记录多少个heap page就认为它有多少个slot。所以slot个数就是叶子节点的个数：

```c
#define SlotsPerFSMPage LeafNodesPerPage
```

可以计算出常见块大小的一些结果：

```
BLCKSZ    NodesPerPage    NonLeafNodesPerPage    LeafNodesPerPage
-----------------------------------------------------------------
 512         480                255                   225
 1024        992                511                   481
 2048        2016               1023                  993
 4096        4064               2047                  2017
 8192        8160               4095                  4065
```

另一方面，PG中每个数据文件（heap或index）中的heap page是有页号的：

```c
typedef uint32 BlockNumber;
#define InvalidBlockNumber      ((BlockNumber) 0xFFFFFFFF)
#define MaxBlockNumber          ((BlockNumber) 0xFFFFFFFE)
```

可以看出，一个数据文件最多有2^32-1个页。

那么引出一个问题，正常情况下一个FSM页无法记录这么多的数据页，所以这个二叉树要成为跨页的二叉树，叫做High-level结构：

![](/images/201909/22.png)

High-level结构中只有最底层PAGE的叶子节点记录的是heap page的空闲空间粒度。上层FSM页的叶子节点记录的是下层FSM页的根节点的值。

那么High-level结构来定位一个heap page，需要三个概念：

* level（层号）
* page number（页号）
* slot（槽号）

为了使算法足够简单，FSM树结构的level应该是一个固定值，4层的话，每页至少需要256个slot才能记录2^32-1个页（256^4>2^32-1），3层的话每页至少需要1626个slot才能记录2^32-1个页（1626^3>2^32-1）。

```c
#define FSM_TREE_DEPTH  ((SlotsPerFSMPage >= 1626) ? 3 : 4)
```

所以4K以上的BLCKSZ都是3层结构。叶子FSM页都是0层，它们的父FSM页在1层，根FSM页是2层。

```c
#define FSM_ROOT_LEVEL  (FSM_TREE_DEPTH - 1)
#define FSM_BOTTOM_LEVEL 0
```

## 源码分析查找过程

* **FSM页内的查找过程在fsm_search_avail函数中**

```c
/*
 * 查找满足条件的slot
 * 返回slot号，没找到返回-1.
 *
 * 调用者必须至少在页上有一个共享锁，另外，如果页需要更新，排他模式下，该函数可以在页上解锁再次加锁。
 * 如果调用者在fsm_search_avail外已经在页上施加排他锁，参数exclusive_lock_held应被设置为true，去避免额外的解锁加锁操作。
 * 如果advancenext是false, fp_next_slot会被设置成返回的返回的slot号，如果是true，设置成返回的slot号的下一个。
 */
int
fsm_search_avail(Buffer buf, uint8 minvalue, bool advancenext, bool exclusive_lock_held)
{
    Page        page = BufferGetPage(buf);                                      //获取页，是个指针
    FSMPage     fsmpage = (FSMPage) PageGetContents(page);                      //获取页中FSM部分
    int         nodeno;
    int         target;
    uint16      slot;
 
restart:
 
    //首先检查根节点，如果她的叶子节点没有足够的空间就快速退出，有没可能根节点不准直接返回-1？有可能，这种情况留到下次vacuum时候去修复。
    if (fsmpage->fp_nodes[0] < minvalue)
        return -1;
 
    //正常FSM以下能保证肯定有节点有足够空间
 
    //从fp_next_slot提示的槽开始搜索。
    target = fsmpage->fp_next_slot;
    //检查一下是明智之举，可能前一次返回-1，或者是最后一个slot
    if (target < 0 || target >= LeafNodesPerPage)
        target = 0;
    //上面的target是slot号，在数组中的位置需要加一个NonLeafNodesPerPage
    target += NonLeafNodesPerPage;
    //这句为什么不和上行代码合并 nodeno = target + NonLeafNodesPerPage 呢？
    nodeno = target;
    //找到有足够空间的节点（她在树中的某个位置）
    while (nodeno > 0)
    {
        //找到符合条件的节点就跳出循环
        if (fsmpage->fp_nodes[nodeno] >= minvalue)
            break;
 
        //当前节点不符合，移动到右边一个节点（或者回卷到同层最左节点），然后找他们的父节点。
        nodeno = parentof(rightneighbor(nodeno));
    }
 
    //从找到的节点开始，自上而下沿着有足够空间的节点路径查找（优先左节点）
    //一直到找到叶子节点才结束，当然发现FSM页不正常会修复重来
    while (nodeno < NonLeafNodesPerPage)
    {
        //获取左子节点位置
        int childnodeno = leftchild(nodeno);
        //左子节点位置小于允许节点总数（理论上最大子位置是NodesPerPage-1），并且左子节点符合条件
        if (childnodeno < NodesPerPage && fsmpage->fp_nodes[childnodeno] >= minvalue)
        {
            //范围缩小到左子节点树，继续循环处理
            nodeno = childnodeno;
            continue;
        }
        //获取右子节点位置
        childnodeno++;
        //右子节点位置小于允许节点总数（理论上最大子位置是NodesPerPage-1），并且右子节点符合条件
        if (childnodeno < NodesPerPage && fsmpage->fp_nodes[childnodeno] >= minvalue)
        {
            //范围缩小到右子节点树，继续循环处理
            nodeno = childnodeno;
        }
        //左右都不满足，说明出现问题了，需要修复，持久话出现页面撕裂会有这种情况。
        //注意如果父节点是7，子节点是3和5，申请5以内的空间都是能正常申请的，只有申请超过5的空间才会触发这个部分代码
        else
        {
            RelFileNode rnode;
            ForkNumber  forknum;
            BlockNumber blknum;
 
            BufferGetTag(buf, &rnode, &forknum, &blknum);
            elog(DEBUG1, "fixing corrupt FSM block %u, relation %u/%u/%u", blknum, rnode.spcNode, rnode.dbNode, rnode.relNode);
 
            //由于需要更新页面，所以要上排他锁，如果已经上了排他锁这部分代码跳过
            if (!exclusive_lock_held)
            {
                //解锁，应该是共享锁
                LockBuffer(buf, BUFFER_LOCK_UNLOCK);
                //上排他锁
                LockBuffer(buf, BUFFER_LOCK_EXCLUSIVE);
                //设置exclusive_lock_held为true，下次循环再出现页面撕裂不重复执行这块代码
                exclusive_lock_held = true;
            }
            //修复页面，从最下面一层非叶子节点的最后一个节点开始往上修复，也就是数组NonLeafNodesPerPage - 1位置节点，往前遍历修复。
            fsm_rebuild_page(page);
            //提示页面被更新
            MarkBufferDirtyHint(buf, false);
            //重新走搜索流程
            goto restart;
        }
    }
 
    //找到最底层的叶子节点位置nodeno，用nodeno减去非叶子节点个数，得到是FSM页内的第几个slot（从0开始）
    slot = nodeno - NonLeafNodesPerPage;
 
    //更新fp_next_slot
    //注意:代码仅仅在持有共享锁的情况下做更新,可能并发情况下fp_next_slot结果有一些混乱,不过代价比独占锁小，毕竟fp_next_slot没那么重要。
    //不用担心下面代码导致fp_next_slot超出LeafNodesPerPage范围，下次开始搜索函数头部会处理这个问题
    //设置了advancenext，会加1个位置
    fsmpage->fp_next_slot = slot + (advancenext ? 1 : 0);
    //返回slot号
    return slot;
}
```

可以看出fp_next_slot是指向了上次搜索到的slot的位置，接下来这个page的每次搜索都会从fp_next_slot标识的位置开始。这样的设定是为了：

* 可以使得不同的backend不至于同时在一个页中搜索导致争抢。
* 在多个backend的访问时可以给多个请求一个尽可能连续的内存空间，有利于操作系统去进行预取（prefetch）和批量写。

* **FSM页间的查找过程在fsm_search函数中**

```c
//查找到合适的heap页，返回heap页号
static BlockNumber
fsm_search(Relation rel, uint8 min_cat)
{
    int         restarts = 0;
    //FSMAddress是（层号，层内部页号），3层结构FSM_ROOT_ADDRESS就是（2，0）
    FSMAddress  addr = FSM_ROOT_ADDRESS;
 
    //由于是high-level可能需要多次读取FSM页，循环处理
    for (;;)
    {
        int         slot;
        Buffer      buf;
        uint8       max_avail = 0;
 
        //从Relation rel的FSM文件读取addr位置的FSM页
        buf = fsm_readbuf(rel, addr, false);
 
        //有效页
        if (BufferIsValid(buf))
        {
            //页上加共享锁
            LockBuffer(buf, BUFFER_LOCK_SHARE);
            //页内查询
            slot = fsm_search_avail(buf, min_cat,(addr.level == FSM_BOTTOM_LEVEL),false);
            //没有符合条件的slot，获取最大可用值，实际返回的就是FSM页树状结构根节点那个byte，用于FSM页间数据不一致的修复工作。
            if (slot == -1)
                max_avail = fsm_get_max_avail(BufferGetPage(buf));
            //解锁
            UnlockReleaseBuffer(buf);
        }
        //无效页
        else
            slot = -1;
 
        //页内找到合适的slot了
        if (slot != -1)
        {
            //沿树装结构自上而下查找
            //如果是最底层页就根据FSM页地址和slot号获取heap页号，并返回。
            if (addr.level == FSM_BOTTOM_LEVEL)
                return fsm_get_heap_blk(addr, slot);
            //不是最底层,获取slot指向的子FSM页的逻辑地址（层号，层内部页号）
            addr = fsm_get_child(addr, slot);
            //继续循环处理
        }
        //无合适的slot，并且是顶层FSM页
        else if (addr.level == FSM_ROOT_LEVEL)
        {
            //最顶层都不满足，底层就不用说了，直接返回无效块号，表示不可以通过FSM机制分配合适的页。
            return InvalidBlockNumber;
        }
        //无合适的slot，并且是非顶层FSM页，说明FSM页之间数据出现了偏差
        else
        {
            //父FSM页逻辑地址和对应到本FSM页的slot
            uint16      parentslot;
            FSMAddress  parent;
 
            /*
             * 对于较低层页, 如果上层页叶子节点值大于下层页根节点值，会触发本块代码。更新上层节点值，并重新开始。
             * 这边有个竞争状态, 在程序A更新本FSM页后释放了本FSM页上的排他锁但是还没有获得父FSM页的排他锁这个时间段内，
             * 另外一个程序B也更新了本FSM页，并先于程序A获取了父FSM页的排他锁，并完成了任务。
             * 显然程序A在程序B释放父FSM页上的排他锁后，会更新父FSM页，显然数据可能是不正确的。
             * 不过这没多大问题，首先这十分罕见，另外在下次vacuum的时候会修复。
             */
            parent = fsm_get_parent(addr, &parentslot);
            fsm_set_and_search(rel, parent, parentslot, max_avail, 0);
 
            //严重过时的上层FSM页可能导致相当多次的修复循环
            //这种页间不一致问题最后都会被修复，但不应该在分配内存的时候做太多次，返回InvalidBlockNumber直接去分配新页，
            //不一致让vacuum去修复。
            if (restarts++ > 10000)
                return InvalidBlockNumber;
 
            //重头开始查找
            addr = FSM_ROOT_ADDRESS;
        }
    }
}
```

## 源码分析更新过程

* **FSM页内更新过程**

```c
//将page的slot节点值设置成value，并向上更新
bool
fsm_set_avail(Page page, int slot, uint8 value)
{
    int         nodeno = NonLeafNodesPerPage + slot;            //获取节点号
    FSMPage     fsmpage = (FSMPage) PageGetContents(page);      //获取FSM页内容
    uint8       oldvalue;
 
    Assert(slot < LeafNodesPerPage);
    //获取旧值
    oldvalue = fsmpage->fp_nodes[nodeno];                      
 
    //如果旧值和要设置的值相等，结束处理
    if (oldvalue == value && value <= fsmpage->fp_nodes[0])
        return false;
    //设置新值
    fsmpage->fp_nodes[nodeno] = value;
 
    //向上扩散，直到根节点，或者某层节点不需要更新为止
    do
    {
        uint8       newvalue = 0;
        int         lchild;
        int         rchild;
 
        nodeno = parentof(nodeno);      //获取父节点
        lchild = leftchild(nodeno);     //父节点左子
        rchild = lchild + 1;            //父节点右子
 
        //获得左右子节点的较大值
        newvalue = fsmpage->fp_nodes[lchild];
        if (rchild < NodesPerPage)
            newvalue = Max(newvalue, fsmpage->fp_nodes[rchild]);
        //获得父节点的值
        oldvalue = fsmpage->fp_nodes[nodeno];
        //两个值相等，无需更新，循环结束
        if (oldvalue == newvalue)
            break;
        //更新父节点值
        fsmpage->fp_nodes[nodeno] = newvalue;
    } while (nodeno > 0);
 
    //保险检查，再check一下，如果FSM页的根节点值比要设置的值小，修复一下这个页
    if (value > fsmpage->fp_nodes[0])
        fsm_rebuild_page(page);
 
    return true;
}
```

* **FSM页间更新过程**

依靠vacuum触发的fsm_vacuum_page函数,就不写了……

# 相关链接

[PostgreSQL Free Space Map Principle](https://github.com/digoal/blog/blob/master/201005/20100511_02.md)

[PostgreSQL FSM(Free Space Map) 源码解读](http://blog.itpub.net/30088583/viewspace-1387176/)

[PgSQL · 原理介绍 · PostgreSQL中的空闲空间管理](http://mysql.taobao.org/monthly/2019/03/06/)

[src/backend/storage/freespace/README](https://github.com/postgres/postgres/blob/master/src/backend/storage/freespace/README)


