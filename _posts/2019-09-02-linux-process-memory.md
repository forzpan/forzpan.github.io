---
title: 进程内存布局
date: 2019-09-02 16:30:00
categories:
- Linux
tags:
- 内存布局
description: Linux系统中进程的内存布局结构
---

# 进程内存布局

![](/images/201909/4.png)

* 代码段：存放指令，是共享的、只读的。可能包括一些只读常量，比如字符串常量。
* 数据段：存放已初始化的全局变量、静态变量、常量数据。
* BBS段： 存放未初始化的全局变量、静态变量。
* 栈段： 在程序运行过程中实时分配和释放，栈由操作系统自动管理。
* 堆段： 堆是由malloc()系列函数分配使用的内存块，堆由程序员管理。

[剖析内存中的程序之秘](https://linux.cn/article-9255-1.html)
