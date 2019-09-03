---
title: CentOS系统配置相关
date: 2019-09-03 01:00:00
categories:
- Linux
tags:
- 运维
description: CentOS系统的一些配置设置
---

* `sysctl -a`可以查看当前所有配置，可以修改`/etc/sysctl.conf`增加对应配置，覆盖默认选项，`sysctl -p`立刻生效。
* 文件`/etc/security/limits.conf`里面配置了对用户的一些限制
* 文件`/etc/profile`和目录`/etc/prfile.d/`下的文件里可配置ulimit参数环境变量什么的
* 每个用户目录下`.bash_profile`也可配置一下设置
* 目录`/etc/rc.local`配置一些启动时初始化操作
* `/etc/fstab`挂载信息，推荐uuid挂载
* `/etc/ld.so.conf.d/`下面可以配置库文件路径，`ldconfig`生效

一个数据库服务进程一般会关心打开文件数。

首先`sysctl -a`可以了解到

* `file-max`决定了当前内核可以打开的最大的文件句柄数。
* `fs.nr_open`是单个进程可分配的最大文件数。数值上又可能比`file-max`大。

`limits.conf`中对应的`nofile`最大上限就是`fs.nr_open`，覆盖掉`fs.nr_open`。

在`/etc/prfile.d/`下会`ulimit -n`会覆盖`limits.conf`配置。

用户目录下`.bash_profile`又会覆盖`/etc/prfile.d/`下的配置文件中的设置。



