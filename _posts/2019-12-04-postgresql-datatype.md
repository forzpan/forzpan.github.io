---
title: PostgreSQL 之 数据类型
date: 2019-12-04 03:00:00
categories:
- PostgreSQL-Base
tags:
- PostgreSQL
description: PostgreSQL中数据类型介绍
---

# 数字类型

## 整数类型

```
smallint        2字节     -32768 to +32767
integer         4字节     -2147483648 to +2147483647
bigint          8字节     -9223372036854775808 to +9223372036854775807
```

## 序数类型

```
smallserial     2字节     1 to 32767
serial          4字节     1 to 2147483647
bigserial       8字节     1 to 9223372036854775807
```

## 任意精度数字

```
decimal         可变      最高小数点前131072位，以及小数点后16383位
numeric         可变      最高小数点前131072位，以及小数点后16383位
```

整数位过多会报错，小数位过多会四舍五入，会靠近远离零的整数，支持输入 NaN

## 浮点数

```
real            4字节     6位十进制精度
double          8字节     15位十进制精度
```

# 字符类型

```
character varying(n), varchar(n)    有限制的变长
character(n), char(n)               定长，空格填充
text                                无限变长
```

没有长度声明词的char等效于char(1)。如果不带长度说明词使用varchar，那么该类型接受任何长度的串。

短值原值存储，长值压缩存储，超长值toast存储。无论怎样，行对应的列原址中最大只能存1G。

```
"char"                              单字节
name                                用于对象名的内部类型，64字节，对象名最大63就是这个限制（有个结束符）
```

# 二进制数据类型

```
bytea        1或4字节外加真正的二进制串        变长二进制串
```

相当于BLOB，底层以二进制格式存储也受到行对应的列原址中最大只能存1G限制。

# 时间和日期类型

```
timestamp [ (p) ] [ without time zone ]     8字节         包括日期和时间，无时区                  4713 BC 到 294276 AD                单位1微秒 / 14位
timestamp [ (p) ] with time zone            8字节         包括日期和时间，有时区                  4713 BC 到 294276 AD                单位1微秒 / 14位
date                                        4字节         日期，没有一天中的时间                  4713 BC 到 5874897 AD               单位1日
time [ (p) ] [ without time zone ]          8字节         一天中的时间，无日期                    00:00:00 到 24:00:00                单位1微秒 / 14位
time [ (p) ] with time zone                 12字节        一天中的时间，不带日期，带有时区         00:00:00+1459 到 24:00:00-1459      单位1微秒 / 14位
interval [ fields ] [ (p) ]                 16字节        时间间隔                              -178000000年 到 178000000年          单位1微秒 / 14位
```

timestamptz被接受为timestamp with time zone的一种简写

time、timestamp和interval接受一个可选的精度值 p，这个精度值声明在秒域中小数点之后保留的位数。缺省情况下，在精度上没有明确的边界，p允许的范围是从 0 到 6。

interval类型有一个附加选项，它可以通过写下面之一的短语来限制存储的fields的集合：

```
YEAR
MONTH
DAY
HOUR
MINUTE
SECOND
YEAR TO MONTH
DAY TO HOUR
DAY TO MINUTE
DAY TO SECOND
HOUR TO MINUTE
HOUR TO SECOND
MINUTE TO SECOND
```

注意如果fields和p被指定，fields必须包括SECOND，因为精度只应用于秒。

# 布尔类型

```
boolean           1字节       状态为真或假
```

`真`状态的有效文字值是：

```
TRUE
't'
'true'
'y'
'yes'
'on'
'1'
```

而对于`假`状态，你可以使用下面这些值：

```
FALSE
'f'
'false'
'n'
'no'
'off'
'0'
```

前导或者末尾的空白将被忽略，并且大小写也无关紧要。使用TRUE和FALSE这样的关键词比较好（SQL兼容）。

# 枚举类型

```
CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');
```

一个枚举类型的值的排序是该类型被创建时所列出的值的顺序。

每一种枚举数据类型都是独立的并且不能和其他枚举类型相比较，即使底层枚举值是一样的，但是可以类型转换成text来比较。

枚举标签是大小写敏感的，因此'happy'是不同于'HAPPY'的。标签内的空白也是有效的。

# 几何类型

```
point       16字节        平面上的点               (x,y)
line        32字节        无限长的线               {A,B,C}
lseg        32字节        有限线段                 ((x1,y1),(x2,y2))
box         32字节        矩形框                   ((x1,y1),(x2,y2))
path        16+16n字节    封闭路径,类似于多边形     ((x1,y1),...)
path        16+16n字节    开放路径                 [(x1,y1),...]
polygon     40+16n字节    多边形,类似于封闭路径     ((x1,y1),...)
circle      24字节        圆                       <(x,y),r>
```

# 网络地址类型

```
cidr        7或19字节      IPv4和IPv6网络
inet        7或19字节      IPv4和IPv6主机以及网络
macaddr     6字节          MAC地址
macaddr8    8字节          MAC 地址 (EUI-64 格式)
```

在对inet或者cidr数据类型进行排序的时候， IPv4 地址将总是排在 IPv6 地址前面，包括那些封装或者是映射在 IPv6 地址里 的 IPv4 地址，例如 `::10.2.3.4` 或者 `::ffff::10.4.3.2`。

inet在一个数据域里保存一个 IPv4 或 IPv6 主机地址，以及一个可选的它的子网。该类型的输入格式是`地址/y`，请注意如果你想只接受网络地址，你应该使用cidr类型而不是inet。

cidr类型保存一个`IPv4`或`IPv6`网络地址声明。其输入和输出遵循无类的互联网域路由（Classless Internet Domain Routing）习惯。声明一个网络的格式是`地址/y`，其中`address`是 IPv4 或 IPv6 网络地址而`y`是网络掩码的位数。

inet和cidr类型之间的本质区别是inet接受右边有非零位的网络掩码， 而cidr不接受。 例如，`192.168.0.1/24`对inet来说是有效的， 但是cidr来说是无效的。

# 位串类型

```
bit(n)
bit varying(n)
```

写一个没有长度的bit等效于 bit(1)，没有长度的bit varying意味着没有长度限制。

# 文本搜索类型

一个tsvector值是一个排序的可区分词位的列表，词位是被正规化合并了同一个词的不同变种的词。

可以有词位和权重。直接类型转换获得tsvector类型值是没有正规化过程的，to_tsvector()函数有正规化过程。

tsquery是用于对tsvector类型值进行查询的，to_tsquery将会以和其他词同样的方式处理前缀。

# json类型

json 和 jsonb。它们 几乎接受完全相同的值集合作为输入。主要的实际区别之一是 效率。json数据类型存储输入文本的精准拷贝，处理函数必须在每 次执行时必须重新解析该数据。而jsonb数据被存储在一种分解好的 二进制格式中，它在输入时要稍慢一些，因为需要做附加的转换。但是 jsonb在处理时要快很多，因为不需要解析。jsonb也支 持索引，这也是一个令人瞩目的优势。

由于json类型存储的是输入文本的准确拷贝，其中可能会保留在语法 上不明显的、存在于记号之间的`空格`，还有 JSON 对象内部的`键的顺序`。还有， 如果一个值中的 JSON 对象包含同一个键超过一次，所有的键/值对都会被保留（ 处理函数会把最后的值当作有效值）。相反，jsonb不保留空格、不 保留对象键的顺序并且不保留重复的对象键。如果在输入中指定了`重复的键`，只有 最后一个值会被保留。

JSON 基本类型和相应的PostgreSQL类型：

```
JSON 基本类型    PostgreSQL类型      注释
string          text                不允许\u0000，如果数据库编码不是 UTF8，非 ASCII Unicode 转义也是这样
number          numeric             不允许NaN 和 infinity值
boolean         boolean             只接受小写true和false拼写
null            (无)                SQL NULL是一个不同的概念
```

关于jsonb的一下操作处理和gin索引应该关注一下

# 数组类型

类似一种同构多值列的实现方式，比较丰富的数组方法可以使用

# 复合类型

类似异构多值列。使用的时候有时候和table有冲突，注意一下

# 范围类型

```
int4range       integer的范围
int8range       bigint的范围
numrange        numeric的范围
tsrange         不带时区的 timestamp的范围
tstzrange       带时区的 timestamp的范围
daterange       date的范围
```

可以为范围类型的表列创建 GiST 和 SP-GiST 索引。

这个对范围的包含，重叠，交集等查询比较方便


