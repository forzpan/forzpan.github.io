---
title: golang 空结构体
date: 2019-09-04 09:00:00
categories:
- GoLang
tags:
- 开发
description: golang中很有用处的空结构体
---

# 介绍

本文详细介绍——空结构体：

```go
type Q struct{}
var q struct{}
```

但是如果一个结构体没有任何字段，不包含任何数据，那么它有什么用处呢？我们如何利用空结构体？

# 宽度

在深入研究空结构体之前，我想先简短介绍一下关于宽度的知识。

宽度这个术语来自于gc编译器，但是他的词源可以追溯到几十年以前。

宽度描述了一个数据类型实例需要占用的字节数，由于进程的内存空间是一维的，我更倾向于将宽度理解为size。

宽度是数据类型的一个属性。Go程序中**几乎**每个值归属一种数据类型，一个值的宽度是由它的数据类型决定的，通常是8bit的整数倍。（写几乎因为 nil这个东西）

我们可以通过unsafe.Sizeof()函数获取任何值的宽度：

```go
var s string
var c complex128
fmt.Println(unsafe.Sizeof(s))    // prints 8
fmt.Println(unsafe.Sizeof(c))    // prints 16
```

数组的宽度是他元素宽度的整数倍。

```go
var a [3]uint32
fmt.Println(unsafe.Sizeof(a)) // prints 12
```

结构体提供了定义组合类型的灵活方式，组合类型的宽度是字段宽度的和，然后再加上填充宽度。

```go
type S struct {
        a uint16
        b uint32
}
var s S
fmt.Println(unsafe.Sizeof(s)) // prints 8, not 6
```

~~上面例子演示了填充的一个特性。就是值的内存对齐，必须是它宽度的整数倍，上面例子中2个字节被填充在a和b之间。~~

**更新** Russ Cox写了个解释，宽度和对齐没什么关系，读[这个](https://dave.cheney.net/2014/03/25/the-empty-struct#comment-2815)。

插一句 [Go unsafe 包之内存布局](https://www.flysnow.org/2017/07/02/go-in-action-unsafe-memory-layout.html) 了解一下。

# 空结构体

现在我们清楚的认识到空结构体的宽度是0，他占用了0字节的内存空间。

```go
var s struct{}
fmt.Println(unsafe.Sizeof(s)) // prints 0
```

由于空结构体占用0字节，那么空结构体也不需要填充字节。所以空结构体组成的组合数据类型也不会占用内存空间。

```go
type S struct {
        A struct{}
        B struct{}
}
var s S
fmt.Println(unsafe.Sizeof(s)) // prints 0
```

# 空结构体可以用来干什么

由于Go的正交性，空结构体可以像其他结构体一样正常使用。正常结构体拥有的属性，空结构体一样具有。

你可以定义一个空结构体组成的数组，当然这个数组不占用内存空间。

```go
var x [1000000000]struct{}
fmt.Println(unsafe.Sizeof(x)) // prints 0
```

空结构体组成的切片的宽度只是它的头部数据的长度，就像上例展示的那样，切片元素不占用内存空间。

```go
var x = make([]struct{}, 1000000000)
fmt.Println(unsafe.Sizeof(x)) // prints 12 in the playground
```

当然切片的衍生切片的长度和容量等属性依旧可以工作。

```go
var x = make([]struct{}, 100)
var y = x[:50]
fmt.Println(len(y), cap(y)) // prints 50 100
```

你甚至可以寻址一个空结构体，空结构体是可寻址的，就像其他类型的实例一样。

```go
var a struct{}
var b = &a
```

有意思的是两个空结构体的地址可以相等。

```go
var a, b struct{}
fmt.Println(&a == &b) // true
```

对于[]struct{}，也具有这样的属性。

```go
a := make([]struct{}, 10)
b := make([]struct{}, 20)
fmt.Println(&a == &b)       // false, a and b are different slices
fmt.Println(&a[0] == &b[0]) // true, their backing arrays are the same
```

为什么会这样？因为空结构体不包含字段，所以不存储数据。如果空结构体不包含数据，那么就没有办法说两个空结构体的值不相等，所以空结构体的值就这样相等了。

```go
a := struct{}{} // not the zero value, a real new struct{} instance
b := struct{}{}
fmt.Println(a == b) // true
```

注意：文档没有说明这个特性，但是有如此表述["Two distinct zero-size variables may have the same address in memory"](http://golang.org/ref/spec#Size_and_alignment_guarantees)

# 空结构体作为作为方法宿主

现在让我们展示一下空结构体如何像其他结构体一样工作的，空结构体可以作为方法的接收者：

```go
type S struct{}
 
func (s *S) addr() { fmt.Printf("%p\n", s) }
 
func main() {
        var a, b S
        a.addr() // 0x1beeb0
        b.addr() // 0x1beeb0
}
```

这个例子中空结构体的地址是0x1beeb0，实际也是所有size为0的值的内存地址，但是这个值可能随着Go版本的不同而发生变化。

# 另外

空结构体一个重要的用途是，用 chan struct{}的形式在不同go程之间发送信号。在[Curious Channels](https://dave.cheney.net/2013/04/30/curious-channels)这篇文章中讨论。

更新：有人提及 [https://github.com/bradfitz/iter/blob/master/iter.go](https://github.com/bradfitz/iter/blob/master/iter.go)，自己看一下好了。

原文连接  [The empty struct](https://dave.cheney.net/2014/03/25/the-empty-struct)


