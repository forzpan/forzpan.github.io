---
title: PostgreSQL 之 配置文件
date: 2019-09-06 02:00:00
categories:
- PostgreSQL-Kernel
tags:
- PostgreSQL
description: PostgreSQL中几个配置文件
---

# pg_hba.conf 主机认证配置文件

记录可以是下面七种格式之一：

```
local      database  user  auth-method  [auth-options]
host       database  user  address  auth-method  [auth-options]
hostssl    database  user  address  auth-method  [auth-options]
hostnossl  database  user  address  auth-method  [auth-options]
host       database  user  IP-address  IP-mask  auth-method  [auth-options]
hostssl    database  user  IP-address  IP-mask  auth-method  [auth-options]
hostnossl  database  user  IP-address  IP-mask  auth-method  [auth-options]
```

空白行和注释行将被忽略。记录不能跨行。空格和\t可以作为分隔符。使用双引号可以包含空白，当all replication这些词被双引号包围，就只表示字面名字意思的对象。

从前往后检查，一旦有符合的行，认证就使用这行，成功或者失败，不会继续往下检查。

* TYPE 认证类型字段
    * `local` 这条记录匹配企图使用 Unix 域套接字的连接。如果没有这种类型的记录，就不允许 Unix 域套接字连接。
    * `host` 这条记录匹配企图使用 TCP/IP 建立的连接。host记录匹配SSL和非SSL的连接尝试。**默认情况下listen_addresses参数是localhost，所以只接受通过本地socket连接**。
    * `hostssl` 这条记录匹配企图使用 TCP/IP 建立的连接，但必须是使用SSL加密的连接。要使用这个选项，编译服务器的时候必须打开SSL支持。此外，必须通过设置ssl配置参数打开SSL。否则，除了记录不能与任何连接匹配的警告外，**hostssl记录将被忽略**。
    * `hostnossl` 这条记录的行为与hostssl相反；它只匹配那些在 TCP/IP上不使用SSL的连接企图。

* DATABASE 字段
    * `all` 匹配所有数据库
    * `sameuser` 匹配请求的用户同名数据库。
    * `samerole` 匹配请求的用户所属角色的同名数据库。
    * `replication` 匹配一个物理复制连接被请求。
    * `数据库名` 匹配具体的数据库，可以逗号分割多个，也可以使用@filename的方式，file里放数据库名。

* USER 字段 
    * `all` 匹配所有用户。
    * `用户名` 特定用户，可逗号分割多个，文件@的方式也可。
    * `+角色名` 匹配这个角色的任何直接或间接成员角色，可逗号分割多个，文件@的方式也可。如果超级用户显式的是一个角色的成员（直接或间接），那么超级用户将只被认为是该角色的一个成员而不是作为一个超级用户。

* ADRESS 字段 （这个域只适用于host、 hostssl和hostnossl记录，local没有）
    * `一个主机名` 匹配对应主机，大小写敏感的。一个以点号（.）开始的主机名声明匹配实际主机名的后缀。
    * `一个IP地址范围` 一个 IP 地址范围以该范围的开始地址的标准数字记号指定，然后是一个斜线（/） 和一个CIDR掩码长度。掩码长度表示客户端 IP 地址必须匹配的高序二进制位位数。在给出的 IP 地址中，这个长度的右边的二进制位必须为零。 在 IP 地址、/和 CIDR 掩码长度之间不能有空白。 172.20.143.89/32用于一个主机， 172.20.143.0/24用于一个小型网络， 10.6.0.0/16用于一个大型网络。0.0.0.0/0表示所有 IPv4 地址。
    * `all` 匹配所有主机
    * `samehost` 匹配任何本服务器自身的 IP 地址，可能有多个网卡。
    * `samenet` 匹配本服务器直接连接到的任意子网的任意地址。

* IP-address和IP-mask组合 （ADRESS字段的替代方案）
    * 255.0.0.0表示 IPv4 CIDR 掩码长度 8，而255.255.255.255表示 CIDR 掩码长度 32。这些域只适用于host、hostssl和hostnossl记录。

* auth-method 字段
    * `trust` 无条件地允许连接。
    * `reject` 无条件地拒绝连接。
    * `scram-sha-256` 执行SCRAM-SHA-256验证以验证用户的密码。
    * `md5` 执行SCRAM-SHA-256或MD5验证以验证用户的密码。
    * `password` 要求客户端提供一个未加密的口令进行认证。因为口令是以明文形式在网络上发送的，所以我们不应该在不可信的网络上使用这种方式。
    * `ident` 通过联系客户端的 ident 服务器获取客户端的操作系统用户名，并且检查它是否匹配被请求的数据库用户名。Ident 认证只能在 TCIP/IP 连接上使用。当为本地连接指定这种认证方式时，将用 peer 认证来替代。
    * `peer` 从操作系统获得客户端的操作系统用户，并且检查它是否匹配被请求的数据库用户名。这只对本地连接可用。
    * `cert`
    * `pam`

* auth-options 字段
    * 在auth-method域的后面，可以是形如name=value的域，它们指定认证方法的选项。

**除了pg_hba.conf文件许可，用户还得需要connect权限**。

# postgresql.conf 系统配置文件

[PostgreSQL 10 参数模板](https://github.com/digoal/blog/blob/master/201805/20180522_03.md)

[PostgreSQL 11 参数模板](https://github.com/digoal/blog/blob/master/201812/20181203_01.md)

postgresql.auto.conf 不可手动编辑，这个 文件保存了通过[ALTER SYSTEM](http://www.postgres.cn/docs/10/sql-altersystem.html)命令提供的设置，覆盖postgresql.conf 的值。alter database、alter role。

# pg_ident.conf  用户名称映射的配置文件


