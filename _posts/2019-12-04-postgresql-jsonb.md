---
title: PostgreSQL 之 jsonb
date: 2019-12-04 01:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: PostgreSQL中非结构化数据存储相关内容
---

# 特性

jsonb数据被存储在解析好的二进制格式中，输入时要稍慢一些，因为需要转换。但是jsonb在查询处理时要快很多，因为不需要解析。

jsonb不保留空格、不保留对象键的顺序并且不保留重复的对象键（只保留最后一个）。

# 包含

```sql
-- 简单的标量/基本值只包含相同的值：
SELECT '"foo"'::jsonb @> '"foo"'::jsonb;
 
-- 右边的数字被包含在左边的数组中：
SELECT '[1, 2, 3]'::jsonb @> '[1, 3]'::jsonb;
 
-- 数组元素的顺序没有意义，因此这个例子也返回真：
SELECT '[1, 2, 3]'::jsonb @> '[3, 1]'::jsonb;
 
-- 重复的数组元素也没有关系：
SELECT '[1, 2, 3]'::jsonb @> '[1, 2, 2]'::jsonb;
 
-- 空数组也OK：
SELECT '[1, 2, 3]'::jsonb @> '[]'::jsonb;
 
-- 右边的数组不会被认为包含在左边的数组中，
-- 即使其中嵌入了一个相似的数组：
SELECT '[1, 2, [1, 3]]'::jsonb @> '[1, 3]'::jsonb;  -- 得到假
 
-- 但是如果同样也有嵌套，包含就成立：
SELECT '[1, 2, [1, 3]]'::jsonb @> '[[1, 3]]'::jsonb;
 
-- 右边具有一个单一键值对的对象被包含在左边的对象中：
SELECT '{"product": "PostgreSQL", "version": 9.4, "jsonb": true}'::jsonb @> '{"version": 9.4}'::jsonb;
 
-- 类似的，这个例子也不会被认为是包含：
SELECT '{"foo": {"bar": "baz"}}'::jsonb @> '{"bar": "baz"}'::jsonb;  -- 得到假
 
-- 包含一个顶层键和一个空对象：
SELECT '{"foo": {"bar": "baz"}}'::jsonb @> '{"foo": {}}'::jsonb;
 
-- 这个数组包含基本字符串值：
SELECT '["foo", "bar"]'::jsonb @> '"bar"'::jsonb;
 
-- 反之不然，下面的例子会报告“不包含”：
SELECT '"bar"'::jsonb @> '["bar"]'::jsonb; -- 得到假
```

可以看出数组写单个元素的时候，实际上相当于在这个元素上加了个`[]`，比如

```sql
SELECT '["foo", "bar"]'::jsonb @> '"bar"'::jsonb;
```

实际相当于

```sql
SELECT '["foo", "bar"]'::jsonb @> '["bar"]'::jsonb;
```

# 存在

```sql
-- 字符串作为一个数组元素存在：
SELECT '["foo", "bar", "baz"]'::jsonb ? 'bar';
 
-- 字符串作为一个对象键存在：
SELECT '{"foo": "bar"}'::jsonb ? 'foo';
 
-- 不考虑对象值：
SELECT '{"foo": "bar"}'::jsonb ? 'bar';  -- 得到假
 
-- 和包含一样，存在必须在顶层匹配：
SELECT '{"foo": {"bar": "baz"}}'::jsonb ? 'bar'; -- 得到假
 
-- 如果一个字符串匹配一个基本 JSON 字符串，它就被认为存在：
SELECT '"foo"'::jsonb ? 'foo';
```

测试一个字符串（以一个text值的形式给出）是否出 现在jsonb值`顶层`的一个对象键或者数组元素中。

# 提示


由于 JSON 的包含是嵌套的，因此一个恰当的查询可以跳过对子对象的显式选择。

例如，我们有一个doc列包含着对象，大部分对象包含着tags域，其中有子对象的数组。

查询会找到其中同时包含`"term":"paris"`和`"term":"food"`的子对象的记录，而忽略任何位于tags数组之外的这类键：

```sql
SELECT doc->'site_name' FROM websites
  WHERE doc @> '{"tags":[{"term":"paris"}, {"term":"food"}]}';
```

可以用下面的查询完成同样的事情：

```sql
SELECT doc->'site_name' FROM websites
  WHERE doc->'tags' @> '[{"term":"paris"}, {"term":"food"}]';
```

但是`后一种`方法`灵活性较差`，并且常常也`效率更低`。

# 索引

GIN 索引可以被用来有效地搜索在大量jsonb文档中出现的键或者键值对。

有两种 GIN操作符类，在性能和灵活性方面做出了不同的平衡。

jsonb的默认 GIN操作符类jsonb_ops 支持使用`@>`、`?`、`?&`、`?|`操作符的查询，例：

```sql
CREATE INDEX idxgin ON api USING gin (jdoc);
```

对于索引idxgin，如下查询：

```sql
-- 可以使用索引
SELECT jdoc->'guid', jdoc->'name' FROM api WHERE jdoc @> '{"company": "Magnafone"}';
 
-- 无法使用索引
SELECT jdoc->'guid', jdoc->'name' FROM api WHERE jdoc -> 'tags' ? 'qui';
-- 但是，使用表达式索引，上述查询也能使用一个索引：
CREATE INDEX idxgintags ON api USING gin ((jdoc -> 'tags'));
-- 另一种查询的方法是利用等效的包含来利用索引
SELECT jdoc->'guid', jdoc->'name' FROM api WHERE jdoc @> '{"tags": ["qui"]}';
```

索引将会存储jsonb对象中每一个键和值的拷贝，表达式索引缩小了范围，只存储表达式定义的子对象的键值。

虽然简单索引的方法更加灵活，但是定向的表达式索引更小并且搜索速度比简单索引更快。

jsonb_ops GIN索引为数据中的每一个键和值创建独立的索引项。例如要索引`{"foo": {"bar": "baz"}}`，jsonb_ops会创建三个索引项分别表示foo、bar和baz。那么做包含查询，它将会查找包含全部这三个项的行。

虽然GIN索引能够相当有效地执行这种AND搜索，它仍然不如等效的jsonb_path_ops搜索那样详细和快速（特别是如果有大量行包含三个索引项中的任意一个时）。


非默认的 GIN操作符类`jsonb_path_ops`只支持索引 `@>` 操作符，例：

```sql
CREATE INDEX idxginp ON api USING gin (jdoc jsonb_path_ops);
```

尽管jsonb_path_ops操作符类只支持用`@>`操作符的查询，但它比起默认操作符类jsonb_ops有更客观的性能优势。

jsonb_path_ops GIN索引只为数据中的每个值创建索引项。事实上，每一个jsonb_path_ops索引项是其所对应的值和键的哈希。

例如要索引`{"foo": {"bar": "baz"}}`，将创建一个单一的索引项，它把foo、bar、baz合并计算哈希值。因此一个查找这个结构的包含查询十分高效。

因此一个jsonb_path_ops索引通常也比一个相同数据上的jsonb_ops要小得多，并且搜索的专一性更好，特别是当查询包含频繁出现在该数据中的键时。

但是也可以看出通过jsonb_path_ops GIN索引是无法找到foo是否作为一个键出现。

jsonb_path_ops的另一个不足是它不会为不包含任何值的JSON结构创建索引项，例如`{"a": {}}`。如果需要搜索包含这样一种结构的文档，它将要求一次全索引扫描，那就非常慢。

[了解更多json相关函数](http://www.postgres.cn/docs/11/functions-json.html)

