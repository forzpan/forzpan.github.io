---
title: bleve介绍（5）
date: 2019-09-05 09:00:00
categories:
- Other
tags:
- 全文索引
description: 全文搜索引擎bleve中结果排序
---

Bleve现在可以让您自定义搜索结果的顺序。

默认行为是按相关性对结果进行排序，并以最高评分结果为先。

# 简单的API

将一种SortBy()方法添加到SearchRequest中。

有两个专门用于定义的特殊字段：

* _id ： 指文件标识符
* _score ： 指由Bleve计算的相关性分数

```go
searchRequest.SortBy([]string{"age", "-_score", "_id"})
```

参数是一个字符串数组。每个字符串引用一个字段的名称。

字段的名称可以用'-'字符作为前缀，这会导致该字段被颠倒（降序）。

项目将首先按第一个字段排序。任何具有相同的值的项目，然后也按下一个字段排序，依此类推。

排序顺序中的所有字段都必须编入索引。

# 高级API

简单的API适用于大多数情况，但也有一些需要高级API。

在这些情况下，您必须导入bleve search子包，并构建一个search.SortOrder对象。

这是一部分search.SearchSort对象。

目前有三种实现可用，SortField，SortScore和SortDocID。

通过手动构建这些结构，可以避免简单API的歧义和局限性。

* Sort by fields which happen to start with -
* Sort by fields which happen to conflict with _id or _score
* Control whether documents missing a field value should sort first or last
* Control which value for multi-value fields should be used for sorting
* Force handling field values as a particular type (defaults to auto)

 


