---
title: CentOS帐号与权限
date: 2019-09-05 03:00:00
categories:
- Linux
tags:
- 运维
description: 说明centos的帐号与权限相关设置
---

看鸟哥书的一些笔记

# /etc/passwd

![](/images/201909/11.png)

可以看见每行使用`:`分隔，共7个部分：

* 帐号名称
* 密码（历史原因，目前都是x，加密后的密码在/etc/shadow）
* UID（0：root；1~499系统帐号；500~用户帐号，具体的看/etc/login.defs 里的设置）
* GID（/etc/group里）
* 帐号描述
* /home目录
* 对应shell

# /etc/shadow

![](/images/201909/12.png)

可以看见每行使用`:`分隔，共9个部分：

* 帐号名称
* 密码
* 最后一次更新密码日期（从1971年1月1日的366开始）
* 基于字段3日期，经过多少天密码不可以再次更新
* 基于字段3日期，经过多少天密码必须再次更新
* 距离字段5日期，还有多少天的时发起密码即将过期提醒
* 基于字段5日期，过期密码多少天内可以暂时正常使用
* 帐号失效期
* 保留字段

所以一个帐号的周期有字段8控制，有效和失效两种状态；一个密码的周期有有效，过期，失效三种状态。

# /etc/group

![](/images/201909/13.png)

可以看见每行使用`:`分隔，共4个部分：

* 组名
* 组密码
* GID
* 加入该组的帐号名（初始组不需要）

# /etc/gshadow

![](/images/201909/14.png)

可以看见每行使用`:`分隔，共4个部分：

* 组名
* 组密码
* 组管理员帐号
* 加入该组的帐号（和/etc/group第4个字段一样）

# 初始组和有效组

一个帐号在`/etc/passwd`中`GID`字段对应的组是初始组，加入的其他组会在`/etc/group`第四个字段中添加该帐号名。初始组和其他组都是有效组。

# 相关命令

```
useradd 相关文件 `/etc/login.defs` , `/etc/default/useradd` 和 `/etc/skel/*`
userdel
usermod
passwd
users
chage
id
finger
chfn
chsh

groupadd
groupdel
groupmems
groupmod
groups
newgrp
gpasswd

su
sudo
visudo 就是 vi /etc/sudoers

w
who
last
lastlog

write 中文发不了

pwck
grpck
chpasswd

chown
chgrp
chmod
umask
```

用户没有目录的x权限，无法进入目录。

如果有目录的r权限，只能看见该目录下文件的类型和文件名，无法看到状态位时间的其他信息，也无法进入该目录下面的目录，无法查看该目录下文件内容。

仅仅有目录的x权限是无法删除目录下的文件的，即使那个文件属于该用户，用户同时具有目录的x和w权限才能在该目录下删除，增加，改名等操作。

文件的rwx是对文件的读写执行功能，文件能不能被删除是用户对这个文件所在目录的x和w权限决定的。

也就是可以这么理解目录里的文件是目录的内容，删除文件，增加文件，改名文件得有这个目录的写权限，来修改这个目录的元数据。

进入一个目录，是目录这个文件提供的一个功能，得有执行权限。

