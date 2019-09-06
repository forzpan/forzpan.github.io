---
title: bleve介绍（6）
date: 2019-09-05 10:00:00
categories:
- Other
tags:
- 全文索引
description: 全文搜索引擎bleve中强大的IndexAlias
---

IndexAlias可以用来搜索一个索引，还可以自动切换底层物理索引。IndexAliases也可以搜索跨多个索引同时搜索并正确合并结果。

内部代码使用了go的interface，所以IndexAlias也是一个Index。这意味着一般情况下，您可以像使用索引一样使用IndexAlias

别名概念还增加了3种方法：

* Add，将添加一个或多个索引到别名。
* Remove将从别名中删除一个或多个索引。
* Swap将自动添加/删除提供的索引。

# 切换索引，零停机

* 创建一个新的索引

* 创建一个IndexAlias指向索引

* 编写你的应用程序来查询IndexAlias

* 稍后，创建一个新的索引

* 调用新索引中传递的IndexAlias上的Swap

您的应用程序将安全切换到搜索新索引

# 搜索多个索引

* 创建多个索引

* 创建指向这些索引的IndexAlias

* 编写你的应用程序来调用IndexAlias上的Search

* 您的应用程序将同时搜索所有底层索引，并将结果合并为一个结果

限制：当和IndexAlias指向多个索引时，并非所有操作都受支持。只有可以在所有子索引上调用的操作以及聚合能够响应才受支持。不支持的操作返回bleve.ErrorAliasMulti。

 


