---
title: bleve介绍（2）
date: 2019-09-05 06:00:00
categories:
- Other
tags:
- 全文索引
description: 全文搜索引擎bleve中的一些Character Filter、Tokenizer、Token Filter、Analyzer介绍
---

分词器（Analyzer）由0个或者多个字符过滤器（Character Filter），1个标记生成器（Tokenizer），0个或者多个标记过滤器（Token Filter）组成。

# 相关字符过滤器（Character Filter）

* Regular Expression

正则表达式字符过滤器，一个正则表达式，一个用来替换匹配中文本的字符数组

* HTML

HTML字符过滤器，识别网页标签，用空格替换它们，内在就是Regular Expression字符过滤器实现的，封装了一下而已

* Zero-width Non-Joiner

零宽不连字字符过滤器，将零宽不连字用空格替换

# 相关标记生成器（Tokenizer）

* Single Token

单一Token标记生成器，返回整个输入作为一个Token Stream

* Letter

字母标记生成器，简单识别Unicode文本中属于字母的词作为一个Token，生成Token Stream

* Regular Expression

正则表达式标记生成器，用正则表达式匹配来分隔文本，生成Token Stream

* Whitespace

空白标记生成器，通过空格制表符等一些空白分割文本，生成Token Stream

* Unicode

Unicode标记生成器，使用segment库分割单词边缘，不基于字典的场合比ICU更适用

* ICU

ICU标记生成器，使用ICU库来分割单词边缘，基于字典的场合比较适用

* Exception

Exception标记生成器，文档看不太明白，待会看代码补全

# 相关标记过滤器（Token Filter）

* Apostrophe

撇号标记过滤器，删除撇号后的所有字符

* Camel Case

驼峰标记过滤器，以驼峰位置分割token

* CLD2

CLD2标记过滤器，使用Compact Language Detection 2库，过滤替换原token中的文本，文本转小写

* Compound Word Dictionary

复合词词典过滤器可以让您提供一个单词词典，它们组合起来形成复合词并让您可以单独编制索引

* n-gram

n-gram标记过滤器从每个输入标记计算n-gram。有两个参数，即最小和最大n-gram长度

* Edge n-gram

边缘n-gram标记过滤器将像n-gram标记过滤器一样计算n-gram，但是所有计算的n-gram都位于一侧（前面或后面）

* Elision

elision过滤器识别并删除以撇号前的内容

* Keyword Marker

关键字标记过滤器将标识关键字并将其标记，包含一个关键词map

* Length

长度过滤器标识太长或太短的token。有两个参数，最小长度和最大长度。Token Stream中删除太长或太短的标记

* Lowercase

小写过滤器，将所有字映射成小写

* Porter Stemmer

Porter stemmer过滤器使用Porter词干提取算法，生成Token Stream

* Shingle

Shingle过滤器从输入令牌流计算多令牌带状疱疹。
例如，the quick brown fox配置了最小和最大长度为2，token流将产生the quick，quick brown和brown fox

* Stemmer

词干标记过滤器将输入词汇并应用于词干处理，生成Token Stream

* Stop Token

停止过滤器配置有应该从Token Stream中移除的Token

* Truncate Token

截断标记过滤器将每个输入标记截断为最大标记长度

* Unicode Normalize

Unicode规范化筛选器将输入项转换为指定的Unicode规范化表单。
受支持的形式是：
&emsp;&emsp;NFC
&emsp;&emsp;NFD
&emsp;&emsp;nfkc
&emsp;&emsp;NFKD

# 相关分词器（Analyzer）

* Keyword

关键词分词器 不对输入Text做任何处理，而是将整个输入Text作为一个Token

* Simple

单纯分词器 对输入Text使用一个Unicode Tokenizer + 一个 Lowercase Token Filters

* Standard

标准分词器 对输入Text使用一个Unicode Tokenizer + 一个 Lowercase Token Filters + 一个 英文Stop Token Token Filters

* Detect Language

检测语言分词器用于检查输入文本，使用启发式确定语言的最佳猜测，并对ISO 639语言代码进行索引

对输入Text使用一个Single Tokenizer + 一个 Lowercase Token Filters + 一个 CLD2 Token Filters

* Language Specific Analyzers

具体的各种语言的分词器




