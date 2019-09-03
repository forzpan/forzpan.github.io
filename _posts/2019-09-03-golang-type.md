---
title: golang 类型说明
date: 2019-09-03 05:00:00
categories:
- GoLang
tags:
- 开发
description: golang类型的一些注意事项
---

# 一个例子开始

```go
package main
 
type T1 []int
type T2 []int
 
func main() {
    var v1 []int
    var v2 []int
    var v3 T1
    var v4 T2
 
    v2 = v1
    v3 = v1
    v2 = v3
    v4 = v3
}
```

思考运行结果。

# 官方手册翻译

>A type determines a set of values together with operations and methods specific to those values.A type may be denoted by a type name, if it has one, or specified using a type literal, which composes a type from existing types. The language predeclares certain type names. Others are introduced with type declarations. Composite types—array, struct, pointer, function, interface, slice, map, and channel types—may be constructed using type literals. Each type T has an underlying type: If T is one of the predeclared boolean, numeric, or string types, or a type literal, the corresponding underlying type is T itself. Otherwise, T's underlying type is the underlying type of the type to which T refers in its type declaration.

>类型确定一组值以及特定于这些值的操作和方法。一个类型可以用一个类型名来表示，或者用已存在类型组合成一个类型字面量来指定。go语言预声明了一些类型名，其他的是通过类型声明引入的。（类型声明分两种：别名声明 和 类型定义）。符合类型—array，struct，pointer，function，interface，slice，map，channel—可以通过类型字面量来构造。每个T类型，都拥有一个基础类型：如果T是预声明的boolean，numeric，string类型，或者是一个类型字面量，它对应的基础类型就是它自己。否则，T的基础类型是T在其类型声明中引用的类型的基础类型。

# 有名类型和无名类型

叫做`Named Type`和`Unnamed Type`，翻译过来够生硬的。

内置类型`bool`，`int`，`int8`，`float`，`float32`，`string`和类型定义`type names []string`，`type salary float32`属于`Named Type`。

`[10]int`，`[]string`，`map[int]string`，`interface{}`，`struct{}`等类型字面量就是`Unnamed Type`。`Unamed type`是基于`Named type`声明的复合类型。

`Named type`可以定义自己的方法（`type Books []string`，可以给`Books`绑定一些方法）。

`Unnamed type`不可以定义自己的方法（即使用`type Books = []string`这种别名声明，也无法给`Books`绑定方法）。

# 基础类型

每个T类型，都拥有一个基础类型

内置类型和类型字面量的基础类型是它本身，比如：

* `bool`的基础类型是`bool`
* `string`的基础类型是`string`
* `[]int`的基础类型是`[]int`
* `struct{}`的基础类型是`struct{}`

否则，T的基础类型是T在其类型声明中引用的类型的基础类型，比如：

```go
type age int
type nianling age
type ages []age
type agearr ages
```

* `age`的引用类型是`int`，`int`的基础类型是自己，所以`age`的基础类型是`int`。
* `nianling`的引用类型是`age`，`age`的基础类型是`int`，所以`nianling`的基础类型是`int`。
* `ages`的引用类型是`[]age`，`[]age`的基础类型是自己，所以`ages`的基础类型是`[]age`。
* `agearr`的引用类型是`ages`，`ages`的基础类型是`[]age`，所以`agearr`的基础类型是`[]age`。

直接用相同`Unnamed type`声明的变量的类型都相同，而对于`Named type`变量而言，即使它们的底层类型相同，它们也是不同类型：

```go
type T1 []int
type T2 []int
 
func main() {
    var v1 []int
    var v2 []int
    var v3 T1
    var v4 T2

    v2 = v1   //ok 两个Unnamed Type
    v3 = v1   //ok Unnamed Type赋值给Named Type
    v2 = v3   //ok Named Type赋值给Unnamed Type
    v4 = v3   //cannot use v3 (type T1) as type T2 in assignment
}
```

`v1`，`v2`是相同`Unnamed Type`变量，`v3`和`v4`是不同`Named Type`变量，两个不同的`Named Type`数据之间是不能互相赋值的。

赋值实际上是一种`copy`，基础类型相同是赋值成为可能的基础，但是`Named Type`类型还可能被绑定了各种方法，这些部分元信息是不可控的，所以go不支持不同的`Named Type`类型之间赋值。

另外 当使用`type`定义一个新类型，它不继承原类型的方法集，如果想继承，就是使用常见的结构体匿名字段的方式。

原文连接 [Go 迷思之 Named 和 Unnamed Types](https://gocn.vip/article/501)


