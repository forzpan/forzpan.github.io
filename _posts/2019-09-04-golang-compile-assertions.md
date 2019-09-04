---
title: golang 编译时断言
date: 2019-09-04 11:00:00
categories:
- GoLang
tags:
- 开发
description: 有意思的golang编译时断言
---

这篇文章是介绍Go中进行编译时断言的一种鲜为人知的方法。你可能不会使用它，但应该知道它很有趣。

Go中有一种相当著名的编译时断言形式：接口符合度检查。

```go
package main
 
import "io"
 
type W struct{}
 
func (w W) Write(b []byte) (int, error)       { return len(b), nil }
func (w W) WriteString(s string) (int, error) { return len(s), nil }
 
type stringWriter interface {
    WriteString(string) (int, error)
}
 
var _ stringWriter = W{} //here
 
func main() {
    var w W
    io.WriteString(w, "very long string")
}
```

很棒，但是如果你想检查一个普通的旧布尔表达式1+1==2怎么办？

比如以下demo：

```go
package main
 
import "crypto/md5"
 
type Hash [16]byte
 
func init() {
    if len(Hash{}) < md5.Size {
        panic("Hash is too small")
    }
}
 
func main() {
    // ...
}
```

怎样让`len(Hash{}) < md5.Size`在运行之前就检查？

可以这样做：

```go
package main
 
import "C"
 
import "crypto/md5"
 
type Hash [16]byte
 
func hashIsTooSmall()
 
func init() {
    if len(Hash{}) < md5.Size {
        hashIsTooSmall()
    }
}
 
func main() {
    // ...
}
```

让go编译器认为hashIsTooSmall是一个C函数，这样在链接时，就会检查 `len(Hash{}) < md5.Size`了。


原文：[compile-time-assertions](https://commaok.xyz/post/compile-time-assertions)

