---
title: golang string和rune
date: 2019-09-04 10:00:00
categories:
- GoLang
tags:
- 开发
description: 分析 golang 中 string 到 rune 发生了什么
---

string  本质是byte数组

rune   本质是int32的别名

```go
package main
 
import (
    "fmt"
    "reflect"
    "unsafe"
)
 
func main() {
    s := "A你好B"
    sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
 
    b := (*[8]byte)(unsafe.Pointer(sh.Data))
    fmt.Printf("%08b\n", b)
 
    bsx := []rune(s)
    bsxh := (*reflect.SliceHeader)(unsafe.Pointer(&bsx))
    y1 := (*[4]rune)(unsafe.Pointer(bsxh.Data))
    fmt.Printf("%032b\n", y1)
    y2 := (*[16]byte)(unsafe.Pointer(bsxh.Data))
    fmt.Printf("%08b\n", y2)
}
```

结果:

![](/images/201909/5.png)

根据utf-8编码规则

|             存储格式                 |     值区间     |
| :---------------------------------- | -------------: | 
| 0xxxxxxx                            | 0-127          | 
| 110xxxxx 10xxxxxx                   | 128-2047       |
| 1110xxxx 10xxxxxx 10xxxxxx          | 2048-65535     |
| 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx | 65536-0x10ffff |

分析一下

![](/images/201909/6.png)

可以看出string在转rune的时候，是根据utf-8编码转去掉规则标志位后连接组成的二进制数，按int32内存分布规则重新分布

