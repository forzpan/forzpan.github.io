---
title: PostgreSQL 之 全文搜索
date: 2019-12-05 03:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: 介绍PostgreSQL中全文搜索功能
---

`全文搜索`提供了确定满足一个`查询`的`自然语言文档`的能力，并可以选择将它们按照与`查询的相关度`排序。

最简单的搜索认为`查询`是一组词而`相似性`是查询词在文档中的频度。

查询对象作为一个文档（大文本），需要被预处理：`将文档解析成记号`（解析器的工作）、`将记号转换成词位`（词典的工作）、`为搜索优化存储预处理好的文档`（附加排名所需的位置和权重信息）。

PG用`tsvector`类型来存储预处理后的文档，`tsquery`类型来表示处理过的查询。

对于文本搜索目的，每一个文档必须被缩减成预处理后的tsvector格式。搜索和排名都在文档的tsvector表示上执行，因此经常把tsvector说成是文档，但是当然它只是完整文档的一种紧凑表示。

PostgreSQL中的全文搜索基于匹配操作符`@@`来完成。

例如：

```sql
SELECT to_tsvector('fat cats ate fat rats') @@ to_tsquery('fat & rat');
```

支持4种类型：

```sql
tsvector @@ tsquery
tsquery  @@ tsvector
text @@ tsquery             -- 等价于 to_tsvector(text) @@ tsquery
text @@ text                -- 等价于 to_tsvector(text) @@ plainto_tsquery(text)
```

tsquery中`AND`、`OR`、`NOT`、`Followed by`等操作符：

```sql
to_tsquery('fat & rat');            -- AND
to_tsquery('fat | rat');            -- OR
to_tsquery('fat & ! rat');          -- NOT
to_tsquery('fatal <-> error');      -- FOLLOWED BY  相当于 <1> 就是要求文档内fatal后面一个词是error
to_tsquery('fatal <2> error');      -- FOLLOWED BY  相当于 <1> 要求fatal后面第二个词是error
to_tsquery('fatal <0> error');      -- FOLLOWED BY  十分特殊的用法，要求两个词在同一位置
```

优先级从高到低是：`!`、`<->`、`&`、`|` 。

表中查询：

```sql
SELECT title
FROM pgweb
WHERE to_tsvector('english', body) @@ to_tsquery('english', 'friend');
```

事实上这种方式非常低效，可以配合GIN索引来加速。

```sql
CREATE INDEX pgweb_idx ON pgweb USING GIN(to_tsvector('english', body)); -- 这里必须明确使用的配置，不能省掉第一个参数。
```

可以建立更复杂的表达式索引，在其中配置名被另一个列指定，例如：

```sql
CREATE INDEX pgweb_idx ON pgweb USING GIN(to_tsvector(config_name, body));
```

还可以定义`tsvector`类型的列来存储处理后的文档。索引直接施加在该列上：

```sql
--增加tsvector类型列
ALTER TABLE pgweb ADD COLUMN textsearchable_index_col tsvector;
--入数据
UPDATE pgweb SET textsearchable_index_col = to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''));
--创建GIN索引
CREATE INDEX textsearch_idx ON pgweb USING GIN(textsearchable_index_col);
--查询
SELECT title
FROM pgweb
WHERE textsearchable_index_col @@ to_tsquery('create & table')
ORDER BY last_mod_date DESC
LIMIT 10;
```

在使用一个单独的列来存储`tsvector`表示时，有必要创建一个触发器在title或body改变时保证tsvector列为当前值。这边需要一个触发器去实现。

---

下面讨论更多细节：

# 解析文档

```sql
SELECT to_tsvector('english', 'a fat  cat sat on a mat - it ate a fat rats');
                  to_tsvector
-----------------------------------------------------
 'ate':9 'cat':3 'fat':2,11 'mat':7 'rat':12 'sat':4
```

`to_tsvector`函数在内部调用一个解析器，把文档文本分解成带类型的记号。对于每个记号，会查询绑定在这个记号类型上的词典列表。第一个识别记号的词典产生一个或多个正规化的词位。

因为to_tsvector(NULL)将返回NULL，当一个域可能为空时，推荐使用coalesce。

# 解析查询

函数`to_tsquery`、`plainto_tsquery`和`phraseto_tsquery`用来把一个查询转换成`tsquery`数据类型。

`to_tsquery`提供了比`plainto_tsquery`和`phraseto_tsquery`更多的特性，但它对输入的要求更加严格。

`to_tsquery`从`querytext`创建一个`tsquery`值，该`querytext`由被`tsquery`操作符`&（AND）`、`|（OR）`、`!（NOT）`和`<->（FOLLOWED BY）`分隔的单个记号组成，

可以使用圆括号分组，使用指定的或者默认的配置把每一个记号正规化成一个词位，并且丢弃掉任何根据配置是停用词的记号。

```sql
--停词省略，前缀和权重匹配
SELECT to_tsquery('The & supern:*A & star:A*B');
        to_tsquery
--------------------------
 'supern':*A & 'star':*AB
 
--单引号短语，如有缩写规则匹配会触发，例子中，一个分类词典含规则supernovae stars : sn
SELECT to_tsquery('''supernovae stars'' & !crab');
  to_tsquery
---------------
 'sn' & !'crab'
 
--如果词典没有匹配的缩写规则，单引号短语相当于&关系
SELECT to_tsquery('''supernovae stars'' & !crab');
           to_tsquery
--------------------------------
 'supernova' & 'star' & !'crab'
```

`plainto_tsquery`将未格式化的文本`querytext`转换成一个`tsquery`值。该文本被解析并被正规化，很像t`o_tsvector`，然后`&（AND）`布尔操作符被插入到留下来的词之间。

```sql
SELECT plainto_tsquery('english', 'The Fat Rats');
 plainto_tsquery
-----------------
 'fat' & 'rat'
```

注意`plainto_tsquery`不会识其输入中的`tsquery`操作符、权重标签或前缀匹配标签：

```sql
SELECT plainto_tsquery('english', 'The Fat:* & Rats:C');
   plainto_tsquery
---------------------
 'fat' & 'rat' & 'c'
```

`phraseto_tsquery`行为和`plainto_tsquery`很像，不过用`<->`操作符插入到留下来的词之间，而不是`&`操作符。还有，停用词也不是简单地丢弃掉，而是通过插入`<N>`操作符来解释。

```sql
SELECT phraseto_tsquery('english', 'The Big Fat to Rats');
     phraseto_tsquery
---------------------------
 'big' <-> 'fat' <2> 'rat'
```

和`plainto_tsquery`相似，`phraseto_tsquery`函数也不会识别其输入中的`tsquery`操作符、权重标签或者前缀匹配标签：

```sql
SELECT phraseto_tsquery('english', 'The Fat:* & Rats:C');
    phraseto_tsquery
-------------------------
 'fat' <-> 'rat' <-> 'c'
```

---

PostgreSQL的文本搜索功能提供了四类配置相关的数据库对象

# 解析器

负责把未处理的文档文本拆分成记号并且标识每个记号的类型（类型集合由解析器本身定义）。解析器完全不会修改文本，只是简单地标识看似有理的词边界。

解析器的一个“字母”的概念由数据库的区域设置决定，具体是LC_CTYPE。

解析器有可能从同一份文本得出相互覆盖的记号。例如，一个带连字符的词可能会被报告为一整个词或者多个部分：

```sql
SELECT alias, description, token FROM ts_debug('foo-bar-beta1');
      alias      |               description                |     token
-----------------+------------------------------------------+---------------
 numhword        | Hyphenated word, letters and digits      | foo-bar-beta1
 hword_asciipart | Hyphenated word part, all ASCII          | foo
 blank           | Space symbols                            | -
 hword_asciipart | Hyphenated word part, all ASCII          | bar
 blank           | Space symbols                            | -
 hword_numpart   | Hyphenated word part, letters and digits | beta1
```

# 词典

对于解析器的输出记号，进行：停用词筛除，复数转单数，同义词替换，抛弃词典列表不能识别的等正规化操作，最终得到词位。

词典接受一个记号作为输入，并返回：
* 如果输入的记号对词典是已知的，则返回一个词位数组（注意一个记号可能产生多于一个词位）。
* TSL_FILTER标志被设置的单一词位，用一个新记号来替换要被传递给后续字典的原始记号（做这件事的字典被称为过滤字典）。
* 如果字典知道该记号但它是一个停用词，则返回一个空数组。
* 如果字典不识别该输入记号，则返回NULL（词典下一个词典会接手这个记号）。

最狭窄的词典放在词典列表的最前面，过滤词典输出仍然是记号，因此不能放在最后位置。

# 词典模板

词典模板提供位于词典底层的函数（一个词典要指定一个模板和一组用于模板的参数）。
* 停用词： 不同词典的停用词处理顺序不一定一样，有的可能优先剔除停用词，有的先正规化，再剔除停用词。
* simple词典模板： 主要是转小写，并根据停用词文件过滤停用词。
* synonym词典模板： 这个词典模板被用来创建用于同义词替换的词典。
* thesaurus词典模板： 同义词词典的一个扩展，并增加了短语支持。
* Ispell词典模板： 词法词典，可以把一个词的很多不同语言学的形式正规化成相同的词位。
* Snowball词典模板： 词干分析词典。Snowball词典识别所有的东西，不管它能不能简化该词，因此应当被放置在词典列表的最后。

# 文本搜索配置

把一个解析器和一组处理解析器输出记号的词典绑定在一起。

对于解析器、词典以及要索引哪些记号类型是由所选择的文本搜索配置决定的。默认文本搜索配置是english。

# 处理流程图

![](/images/201909/27.png)

按自己理解画了一个^_^

# 配置的例子

```sql
--创建一个配置
CREATE TEXT SEARCH CONFIGURATION public.pg ( COPY = pg_catalog.english );
--定义同义词词典
CREATE TEXT SEARCH DICTIONARY pg_dict (
    TEMPLATE = synonym,
    SYNONYMS = pg_dict
);
--注册Ispell词典
CREATE TEXT SEARCH DICTIONARY english_ispell (
    TEMPLATE = ispell,
    DictFile = english,
    AffFile = english,
    StopWords = english
);
--绑定记号类型和词典
ALTER TEXT SEARCH CONFIGURATION pg
    ALTER MAPPING FOR asciiword, asciihword, hword_asciipart, word, hword, hword_part
    WITH pg_dict, english_ispell, english_stem;
--选择不索引或搜索某些内建配置的记号类型
ALTER TEXT SEARCH CONFIGURATION pg
    DROP MAPPING FOR email, url, url_path, sfloat, float;
--测试自定义配置
SELECT * FROM ts_debug('public.pg', '
PostgreSQL, the highly scalable, SQL compliant, open source object-relational
database management system, is now undergoing beta testing of the next
version of our software.
');
```

# 排名搜索结果

排名处理尝试度量文档和特定查询的接近程度,比如考虑查询词在文档中出现的频率，密度，以及所在文档部分的重要程度，甚至文档的创建，更新时间，可以看出需要一个量化模型，一般是根据业务定制的。

通常权重被用来标记来自文档特别区域的词，比如正文和标题。权重标签是应用到位置而不是词位。strip函数会剥离tsvector中的位置和权重信息，只留下词位向量。

setweight函数可用来对tsvector中的项标注一个给定的权重，可选权重是：A、B、C、D（1.0, 0.4, 0.2, 0.1）四个字母之一。这种信息可以被用来排名搜索结果。

目前有两种内建的排名函数，可以作为基础或者样例开发适合业务的排名函数：

```sql
--基于向量的匹配词位的频率来排名
ts_rank([ weights float4[], ] vector tsvector, query tsquery [, normalization integer ]) returns float4
--基于向量的匹配词位的频率和查询覆盖密度来排名
ts_rank_cd([ weights float4[], ] vector tsvector, query tsquery [, normalization integer ]) returns float4
```

排名可能会非常高的代价，因为它要求查询每一个匹配文档的tsvector，这可能会涉及很多I/O而很慢。

# 用于自动更新的触发器

当一个列存文档，一个列存tsvector时，为了保持两列一致，需要一个触发器。PG内置了两个，但是对所有列是一视同仁的，需要不同权重，需要自定义触发器：

```sql
tsvector_update_trigger(tsvector_column_name, config_name, text_column_name [, ... ])
tsvector_update_trigger_column(tsvector_column_name, config_column_name, text_column_name [, ... ])
```

config_name必须是模式限定的，保证触发器不会因为search_path改变而改变。

# 收集文档统计数据

```sql
ts_stat(sqlquery text, [ weights text, ] OUT word text, OUT ndoc integer, OUT nentry integer) returns setof record
```

* sqlquery 是一个文本值，它包含一个必须返回单一tsvector列的 SQL 查询。
* word 是一个词位的值。
* ndoc 词出现过的文档（tsvector）的数量。
* nentry 词出现的总次数。

# 索引加速

全文搜索并非一定需要索引，但GIN（通用倒排索引）和GiST（通用搜索树）索引可以被用来加速全文查询。

GIN索引只支持tsvector类型的列。作为倒排索引，每个词位都有一个索引项（压缩过的匹配位置的列表），多词搜索是找到第一个匹配，然后使用该索引移除缺少额外词的行。

GIN索引只存储词位，并不存储权重标签。涉及权重查询需要回表检查。

GiST索引是有损的，可能产生假匹配，原理类似布隆过滤器，PostgreSQL在需要时会自动做回表检查，消除假匹配，通过词典减少唯一词的数量可以使映射位串冲突减小，降低假匹配概率。

GIN索引的构建时间常常可以通过增加maintenance_work_mem来改进，而GiST索引的构建时间则与该参数无关。

# 限制

一个tsvector（词位和位置）的长度必须小于 1 兆字节。

一个tsquery中结点（词位或操作符）的个数必须小于32768。

每一个词位的长度必须小于2K 字节。

词位的数量必须小于264。

每个词位不超过256个位置。

tsvector中的位置值必须大于0并且小于16383。

`<N>（FOLLOWED BY）`tsquery操作符中的匹配距离不能超过16384。

# 加亮结果

（这块没搞明白）

函数ts_headline来实现这个功能：

```sql
ts_headline([ config regconfig, ] document text, query tsquery [, options text ]) returns text
```

返回文本document使用查询query之后的原文档摘要。

options的可选项：
* StartSel、StopSel：用来定界摘要中出现的查询词，目的是把它们与其他词区分开。
* MaxWords、MinWords：headline包括的最大和最小的单词数量。
* ShortWord：headline的头部和尾部，少于等于这个数值的词位会被删除。
* HighlightAll：布尔标志，如果为true整个text将被用作摘要，并忽略前面的三个参数。
* MaxFragments：要显示的文档摘要或片段的最大数量。默认值0，使用非面向片段的headline生成方法。大于0使用基于片段的headline生成方式，就是找到有尽可能多查询词的文本片段然后往两边扩展片段范围，直到这个片段达到MinWords和MaxWords限定的字数，当然这个片段的头部和尾部定义为ShortWord的单词会被移除。如果文档中没有找到所有的查询词，第一个MinWords字数的片段会被展示。
* FragmentDelimiter：当多于一个片段被显示时，片段将被这个字符串所分隔。比如 "..."

默认值如下：

```sql
StartSel=<b>, StopSel=</b>, MaxWords=35, MinWords=15, ShortWord=3, HighlightAll=FALSE, MaxFragments=0, FragmentDelimiter=" ... "
```
