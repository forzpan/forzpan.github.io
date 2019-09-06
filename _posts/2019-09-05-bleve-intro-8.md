---
title: bleve介绍（8）
date: 2019-09-05 11:00:00
categories:
- Other
tags:
- 全文索引
description: 全文搜索引擎bleve中源码目录分析
---

* bleve 顶级bleve 为所有较低级别的软件包提供易于使用的包装。
* analysis 包含与分析文本相关的所有代码. 通常这个包是独立于其他的。不应该依赖于索引或搜索包。
    * analyzer 包含预制分词器以供一般用途使用。
    * char 包含CharFilter接口的实现。
    * datetime 包含DateTimeParser接口的实现。
    * lang 包含用于特定语言分析的分词器包。
    * token 包含TokenFilter接口的实现。
    * tokenizer 包含Tokenizer接口的实现。
    * tokenmap 支持维护单词或Token列表。
* cmd 命令行目录
* config 功能配置
* document 包含与bleve文档和字段相关的代码。文件包含字段。这是bleve中的索引单位。
* index 包含了所有与在磁盘上放置位相关的代码，以便稍后进行搜索。
* geo 地理信息索引
* http 一组可选的HTTP处理程序，通过HTTP/JSON公开bleve功能。
* index 包含与索引相关代码
    * scorch
    * store 定义了一个普通的KV商店界面。该接口允许索引实现轻松插入替代KV商店。
    * upside_down 倒排索引实现。它可以使用任何存储实现。这具有围绕单个行如何编码的所有细节。
* registry 为应用程序通过字符串名称引用搜索组件提供了便利的机制。这也便于序列化索引映射并将它们与索引一起保存。
* search 搜索包中包含了实现搜索功能的所有代码。取决于Index包暴露的接口，但不应该依赖于它的任何实现细节。
    * collector 负责收集所有结果中的理想结果。通常情况下，按照某些标准排名前n
    * facet 负责构建结果集中的构面信息。
    * highlight 负责在搜索结果中生成突出显示的匹配文本。
    * scorer 负责对搜索结果点击进行评分。这些结果可能是最终或中间结果。
    * searcher 软件包包含实际的搜索者实现。


