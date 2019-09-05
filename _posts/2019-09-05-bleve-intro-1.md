---
title: bleve介绍（1）
date: 2019-09-05 05:00:00
categories:
- Other
tags:
- 全文索引
description: 全文搜索引擎bleve中的一些全文术语
---

核心是Analyzer，一般翻译成分词器：

* **Analyzer** 

分词器（Analyzer）是将输入文本（Text）最终转换成一个标记段流（Token Stream）的类似管道的结构，或者说处理流程。

由0个或者多个字符过滤器（Character Filter），1个标记段生成器（Tokenizer），0个或者多个标记段过滤器（Token Filter）组成。

Analyzer 构成：

> Text  ->  Character Filter[0,n]  ->  Term  ->  Tokenizer  ->  Token Stream  ->  Token Filter[0,n]  ->  Token Stream

流程很清晰，下面依次解释涉及的术语：

* **Term**

词条（Term）就是一串unicode编码的字符。

* **Text**

文本（Text）就是一个常规的词条（Term），即一串unicode编码的字符，但一般表示未处理数据。

* **Token**

标记段（Token）是在文档或字段中特定位置出现的词条（Term）。

* **Token Stream**

标记段流（Token Stream）就是一堆标记段（Token）。

* **Character Filter**

字符过滤器（Character Filter），处理输入文本，一般用来过滤掉没有价值的字符，做点其他的处理也不是不可以。

* **Tokenizer**

标记段生成器（Tokenizer）将输入文本（Text） 分割成一个或者多个标记段（Token），经典的场景，比如把一句话分割成好多词。

* **Token Filter**

标记段过滤器（Token Filter）处理标记段流（Token Stream）中每个标记段（Token）产出另外一个标记段流（Token Stream）。

结果可能是原来的标记段流（Token Stream），也可能多了、少了或者改变了某些标记段（Token）。

 


