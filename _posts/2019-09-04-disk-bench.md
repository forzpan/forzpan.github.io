---
title: SATA,SAS,SSD 性能测试
date: 2019-09-04 01:00:00
categories:
- Linux
tags:
- 运维
description: 对比SATA,SAS,SSD的性能
---

[原文连接](https://blog.csdn.net/killmice/article/details/42745937)

# 参考表

磁盘  | 顺序读   |  顺序写  |   随机读   |  随机写
:--: | -------: | -------: | --------: | ------:
SATA | 124.1M/s | 124.9M/s | 114 iops  | 134 iops
SAS  | 190M/s   | 190M/s   | 456 iops  | 512 iops
SSD  | 404M/s   | 592M/s   | 129K iops | 140K iops


# 顺序读

> fio -name iops -rw=read -bs=4k -runtime=60 -iodepth 32 -filename /dev/sda1 -ioengine libaio -direct=1

* [SATA] Jobs: 1 (f=1): [R] [16.4% done]  [124.1M/0K /s] [31.3K/0  iops] [eta 00m:51s]
* [SAS]  Jobs: 1 (f=1): [R] [16.4% done]  [190M/0K /s]   [41.3K/0  iops] [eta 00m:51s]
* [SSD]  Jobs: 1 (f=1): [R] [100.0% done] [404M/0K /s]   [103K /0  iops] [eta 00m:00s]

# 顺序写

> fio -name iops -rw=write -bs=4k -runtime=60 -iodepth 32 -filename /dev/sda1 -ioengine libaio -direct=1

* [SATA] Jobs: 1 (f=1): [W] [21.3% done]  [0K/124.9M /s] [0 /31.3K iops] [eta 00m:48s]
* [SAS]  Jobs: 1 (f=1): [W] [21.3% done]  [0K/190M /s]   [0 /36.3K iops] [eta 00m:48s]
* [SSD]  Jobs: 1 (f=1): [W] [100.0% done] [0K/592M /s]   [0 /152K  iops] [eta 00m:00s]

# 随机读

> fio -name iops -rw=randread -bs=4k -runtime=60 -iodepth 32 -filename /dev/sda1 -ioengine libaio -direct=1

* [SATA] Jobs: 1 (f=1): [r] [41.0% done]  [466K/0K /s]  [114 /0  iops]  [eta 00m:36s]
* [SAS]  Jobs: 1 (f=1): [r] [41.0% done]  [1784K/0K /s] [456 /0  iops]  [eta 00m:36s]
* [SSD]  Jobs: 1 (f=1): [R] [100.0% done] [505M/0K /s]  [129K /0  iops] [eta 00m:00s]

# 随机写

> fio -name iops -rw=randwrite -bs=4k -runtime=60 -iodepth 32 -filename /dev/sda1 -ioengine libaio -direct=1

* [SATA] Jobs: 1 (f=1): [w] [100.0% done] [0K/548K /s]  [0 /134  iops]  [eta 00m:00s]
* [SAS]  Jobs: 1 (f=1): [w] [100.0% done] [0K/2000K /s] [0 /512  iops]  [eta 00m:00s]
* [SSD]  Jobs: 1 (f=1): [W] [100.0% done] [0K/549M /s]  [0 /140K  iops] [eta 00m:00s]

