---
title: golang 唯一id库
date: 2019-09-03 06:00:00
categories:
- GoLang
tags:
- 开发
description: golang的几个唯一ID库
---

目的是想要在非协调的状态下，获取一个唯一id。

一个要的唯一id，有些特点：

* 唯一的
* 可以按字串顺序排序
* 按时间聚类，时间接近的排序靠近
* 字符串表示可以用作URL的一部分而不进行转义
* 越短越好

Go中有以下实现：

Package | Id | Format
------- | -- | ------
github.com/segmentio/ksuid | 0pPKHjWprnVxGH7dEsAoXX2YQvU | 4 bytes of time (seconds) + 16 random bytes
github.com/rs/xid | b50vl5e54p1000fo3gh0 | 4 bytes of time (seconds) + 3 byte machine id + 2 byte process id + 3 bytes random
github.com/kjk/betterguid | -Kmdih_fs4ZZccpx2Hl1 | 8 bytes of time (milliseconds) + 9 random bytes
github.com/sony/sonyflake | 20f8707d6000108 | ~6 bytes of time (10 ms) + 1 byte sequence + 2 bytes machine id
github.com/oklog/ulid | 01BJMVNPBBZC3E36FJTGVF0C4S | 6 bytes of time (milliseconds) + 8 bytes random
github.com/chilts/sid | 1JADkqpWxPx-4qaWY47~FqI | 8 bytes of time (ns) + 8 random bytes
https://github.com/lithammer/shortuuid | dwRQAc68PhHQh4BUnrNsoS | UUIDv4 or v5, encoded in a more compact way
github.com/satori/go.uuid | 5b52d72c-82b3-4f8e-beb5-437a974842c | UUIDv4 from RFC 4112 for comparison
https://github.com/google/uuid | c01d7cf6-ec3f-47f0-9556-a5d6e9009a43 | UUIDv4

其中
* ksuid生成很快。
* xid是mongodb选用的。
* ulid支持自定义随机算法，可以适配复杂api。
* sonyflake基于Twitter的设计,最小。


[原文链接](https://blog.kowalczyk.info/article/JyRZ/generating-good-random-and-unique-ids-in-go.html)

