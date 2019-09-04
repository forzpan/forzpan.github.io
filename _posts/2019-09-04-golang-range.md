---
title: golang range循环内部
date: 2019-09-04 12:00:00
categories:
- GoLang
tags:
- 开发
description: 深入分析golang range循环内部
---

# 一个疑惑和三个例子

下面的程序会无限执行下去吗？

```go
func main() {
    v := []int{1, 2, 3}
    for i := range v {
        v = append(v, i)
    }
}
```

[答案](https://play.golang.org/p/38-xQZk5luC)

思考一下下面三个例子的结果。

```go
func IndexArray() {
    a := [...]int{1, 2, 3, 4, 5, 6, 7, 8}
 
    for i := range a {
        a[3] = 100
        if i == 3 {
            fmt.Println("IndexArray", i, a[i])
        }
    }
}
 
func IndexValueArray() {
    a := [...]int{1, 2, 3, 4, 5, 6, 7, 8}
 
    for i, v := range a {
        a[3] = 100
        if i == 3 {
            fmt.Println("IndexValueArray", i, v)
        }
    }
}
 
func IndexValueArrayPtr() {
    a := [...]int{1, 2, 3, 4, 5, 6, 7, 8}
 
    for i, v := range &a {
        a[3] = 100
        if i == 3 {
            fmt.Println("IndexValueArrayPtr", i, v)
        }
    }
}
```

[答案](https://play.golang.org/p/uYbJ5KcSjAq)

# range左值

可以是`i := range`或者`i,v := range`，变量已经被预先定义的话，这里`:=`当然可以是`=`。

要注意的是 用`:=`时，前面变量是被重用的，想当于第一次用的是`:=`后续循环用的是`=`。

# range右值

可以是以下类型，或者返回以下类型的表达式

* array 或 array的指针
* slice
* string
* map
* 可从中接收的channel

表达式会在loop开始前计算。

如果只是`i := range arr`或者`i := range *arr`这种只要数组索引的循环，由于数组是不可变的，因此在编译时候，可能直接计算`len(arr)`，编译器直接用这个常量结果优化掉range。

# 分析数据类型

在go中，每次分配都伴随copy，比如赋值，函数参数等。

以下几个类型的底层数据结构：

![](/images/201909/7.png)

所以将一个数组分配给一个array，内存级别是整个array的一次复制。

string和slice的分配只是这两个类型头，或者叫做类型元信息结构体的复制，底层数组是没变化的。

map和channel也一样，只是指针的复制。

# 编译器实现

range本质是for语句的一个语法糖。

![](/images/201909/8.png)

编译器会在循环开始前copy一次循环对象。编译器的编译后的逻辑：

array：

![](/images/201909/9.png)

slice：

![](/images/201909/10.png)

可以看出数组是值类型会被整个复制一遍，大数组直接循环代价不小。应该使用数组指针来做循环对象，复制一个指针还是代价小的。

# 关于map一个重要的信息

如果循环过程中，map的一个key-value没有被遍历到就被删除了，这个值在后续的遍历中不会出现。

遍历过程中一个key-value被创建了，继续遍历可能会遍历到这个key-value，也可能被跳过。这个应该涉及map底层的分桶实现。以后有机会介绍一下。

**总之，把range理解成一个函数，被遍历的对象当作range的参数，这样理解或许更好。**

相关链接：[Go Range Loop Internals](https://garbagecollected.org/2017/02/22/go-range-loop-internals)


