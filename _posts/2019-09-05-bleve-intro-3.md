---
title: bleve介绍（3）
date: 2019-09-05 07:00:00
categories:
- Other
tags:
- 全文索引
description: 全文搜索引擎bleve中的IndexMapping简介
---

简单来说，IndexMapping描述了如何对数据模型进行索引。

# 默认IndexMapping

要获得默认的IndexMapping，只需调用：

```go
indexMapping := bleve.NewIndexMapping()
```

IndexMappings包含您希望支持的每种不同类型文档的DocumentMappings。此外，它包含一个DefaultDocumentMapping，它将用于任何没有明确map的类型。

# 文档类型（Document Type）

bleve如何知道文档是什么类型？

* 如果你的对象实现了bleve.Classifier接口，那么bleve将使用由它的Type()方法返回的字符串
* IndexMapping有一个叫做TypeField的设置。您可以将其设置为任何文档路径，并且如果该路径中的值是字符串，则该值将用作Type Field。如果您未自定义此设置，则默认设置为“_type”
* 如果不能从(1)或(2)中确定类型，则将该类型设置为IndexMapping DefaultType。如果您未自定义此设置，则默认设置为“_default”

# DocumentMappings

现在我们看到bleve将如何确定类型，我们可以为我们感兴趣的每种类型提供自定义的DocumentMapping。

假设我们有一个叫做文档类型blog。我们可以为此类型构建DocumentMapping并配置IndexMapping以使用它：

```go
blogMapping := bleve.NewDocumentMapping()
indexMapping.AddDocumentMapping("blog", blogMapping)
```

通过设置DefaultMapping字段，我们还可以设置一个通用映射，该映射将用于任何没有明确映射的类型。

# FieldMappings

文档是分层的并包含命名字段。这些字段可以是值或嵌套的子文档。

我们通过为其设置DocumentMapping来自定义命名字段的行为。

一旦我们为命名字段创建了DocumentMapping，我们可以为其添加0个或更多的FieldMappings。

FieldMappings描述了我们如何解释该字段以及我们想要插入到索引中的内容。

假设我们的博客文档有一个字符串字段name，我们希望使用英文分析器来处理这个字段。

```go
nameFieldMapping := bleve.NewTextFieldMapping()
nameFieldMapping.Analyzer = "en"
blogMapping.AddFieldMappingsAt("name", nameFieldMapping)
```

现在让我们的博客文档有一个嵌套的结构来描述author字段字段name和email。

这一次我们想索引，但不存储author的name。我们希望从_all字段中排除email。

```go
author := bleve.NewDocumentMapping()
authorNameFieldMapping := bleve.NewTextFieldMapping()
authorNameFieldMapping.Store = false
author.AddFieldMappingsAt("name", authorFieldNameMapping)
authorEmailFieldMapping := bleve.NewTextFieldMapping()
authorEmailFieldMapping.IncludeInAll = false
author.AddFieldMappingsAt("email", authorEmailFieldMapping)
blog.AddSubDocumentMapping("author", author)
```

这显示了FieldMapping中其他一些标志的使用。列表如下：

* 索引 - 索引此字段，默认为true
* 存储 - 存储此字段，默认为true
* IncludeTermVectors - 包含此字段的Term向量，默认为true
* IncludeInAll - 在名为_all字段中包含此字段，默认为true

# Text Field 特定选项

可以在字段上使用分析器

可以在多层级配置默认分析器,底层覆盖上层

DocumentMapping 和 IndexMapping 都可以配置默认分析器

# Date Field 特定选项

DateFormat - 一个DateTimeParser的名称，用于分析存储为字符串的日期

您可以在IndexMapping对象中配置DefaultDateTimeParser

# 了解默认类型与默认映射

当Bleve找不到特定文档的类型时，它会自动分配DefaultType。

一旦Bleve确定了类型，它就会查找与此类型名称匹配的DocumentMapping。如果此类型没有明确配置的DocumentMapping，则使用DefaultMapping。

DefaultType将默认为“_default”，并且DefaultMapping默认为默认的DocumentMapping。

考虑啤酒搜索示例应用程序中的一个示例。该映射描述了两种类型的“啤酒”和“啤酒厂”。对于每一个都提供了一个明确的DocumentMapping。

如果您尝试索引缺少类型字段的文档，则会为其分配类型“_default”。然后Bleve会查看是否有为“_default”配置的映射。没有，所以Bleve继续使用DefaultMapping。



