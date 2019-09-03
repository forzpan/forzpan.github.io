---
title: 数据库原型系统
date: 2019-09-03 02:00:00
categories:
- Other
tags:
- 数据库原型
- monetdb
- peloton
- cstore
description: 几个实验性的数据库原型系统
---

# concurrency control/logging recovery

* [MIT phd xiangyao 针对transaction processing workload写的dbx1000](https://github.com/yxymit/DBx1000) 

* [与xiangyao做的类似，不过实现了更多的cc protocol，支持concurrent index](https://github.com/cavalia/cavalia)

* [希望测试transaction性能的话可以考虑用Silo](https://github.com/stephentu/silo)

# distributed transaction

* [基于xiangyao的dbx1000做的distributed database prototype](https://github.com/mitdbg/deneva)

# OLAP query/join

* [eth的implementation](https://www.systems.ethz.ch/projects/paralleljoins)

# 其他

* [monetdb](https://www.monetdb.org/Home)

* [peloton](https://github.com/cmu-db/peloton)

* [cstore](http://db.csail.mit.edu/projects/cstore)



