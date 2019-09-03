---
title: const char*, char const* and char*const
date: 2019-09-02 15:40:00
categories:
- CLang
tags:
- C语言
description: 记录一下 const char*, char const* 和 char*const 的区别
---

把一个声明从右向左读，`*`读成`pointer to`：

```c
char * const cp;    // 读成 cp is a const pointer to char
char const * p;     // 读成 p is a pointer to const char
const char * p;     // 和上一行是同样的意义
```

一些例子：

```c
char x;
    char const cx = x;
        char const * pcx = &cx;
            char const * const cpcx = pcx;
            char const * * ppcx = &pcx;
    char * px = &x;
        char * const cpx = px;
            char * const * pcpx = cpx;
        char * * ppx = &px;
            char * * const cppx = ppx;
            char * * * pppx = &ppx;
```

注意：

```c
const int * * p;
// 相当于
int const * * p;
// 不是
int * * const p;
// 也不是
int * const * p;
```


