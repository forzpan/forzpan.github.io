---
title: bleve介绍（7）
date: 2019-09-05 11:00:00
categories:
- Other
tags:
- 全文索引
description: 全文搜索引擎bleve中一些其他特性
---

# 忽略/禁用部分文档

有时文档包含仅仅想忽略的部分内容。让我们想象我们有以下结构：

```go
type Person struct {
    Name string
    Addr Address
}
```

而且，我们已经决定不想索引或存储任何地址数据。我们可以通过使用以下DocumentMapping来完成此操作。

```go
personMapping := bleve.NewDocumentMapping()
addressMapping := bleve.NewDocumentDisabledMapping()
personMapping.AddSubDocumentMapping("Addr", addressMapping)
```

# 在结果中高亮显示匹配

搜索的一个流行功能是能够在搜索结果中突出显示文档的匹配部分。

**要求：想突出显示的字段必须同时srored和IncludeTermVectors**。必须存储以便我们可以访问原始内容以突出显示，并且必须包含术语向量，以便我们知道每个词条在原始文本内发生的字节偏移量。

考虑我们已经有了以下搜索请求：

```go
query := bleve.NewQueryStringQuery(...)
searchRequest := bleve.NewSearchRequest(query)
```

我们可以使用默认突出显示高亮显示结果：

```go
searchRequest.Highlight = bleve.NewHighlight()
```

或者我们可以请求使用特定的指定高亮器：

```go
searchRequest.Highlight = bleve.NewHighlightWithStyle(highlighterName)
```

没有任何其他配置，它会自动尝试突出显示任何匹配的字段。有时我们只关注突出特定字段或一组字段：

```go
searchRequest.Highlight.AddField(field)
```

高亮器

* HTML高亮器用于网页高亮
* ANSI高亮器用于命令行高亮

# 在结果中包含构面（Facet）

构面允许包含有关与查询匹配的文档的汇总信息。

## Terms Facet

词条构面需要一个名字和大小

```go
stylesFacet := bleve.NewFacetRequest("style", 3)
searchRequest.AddFacet("styles", stylesFacet)
```

构面styles将字段style的每个Term分桶统计，得到top3

## Numeric Range Facet

数字范围构面

```go
var lowToMidAbv = 3.5
var midToHighAbv = 9.0
abvFacet := bleve.NewFacetRequest("abv", 3)
abvFacet.AddNumericRange("low", nil, &lowToMidAbv)
abvFacet.AddNumericRange("medium", &lowToMidAbv, &midToHighAbv)
abvFacet.AddNumericRange("high", &midToHighAbv, nil)
searchRequest.AddFacet("abv", abvFacet)
```

将abv的值分到3个桶

## DateTime Range Facet

日期范围构面

```go
var cutOffDate = time.Now().Add(-365*24*time.Hour)
updatedFacet := bleve.NewFacetRequest("updated", 2)
updatedFacet.AddDateTimeRange("old", time.Unix(0, 0), cutOffDate)
updatedFacet.AddDateTimeRange("new", cutOffDate, time.Unix(999999999, 999999999))
searchRequest.AddFacet("updated", updatedFacet)
```

将updated的日期分到2个桶

## Facet Results

构面的结果

对于每个构面，将返回一个FacetResult，其中包含以下内容：

* 字段： 构面的字段的名称
* 总计： 遇到的值的总数（如果每个文档有一个词语，这应该与搜索结果中的文档总数相匹配）
* 缺失： 此字段没有任何值的文档数量
* 其他： 存在值的文档数量，但不在要求的前N个构面的桶中
* 构面数组： 每个构面包含表示此面片范围/存储区中项目数的计数
    * Term 词条构面包含词条的名称
    * Numeric Range 数字范围构面包含该桶的数值范围
    * DateTime Range 日期范围构面包含该桶的日期范围