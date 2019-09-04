---
title: golang Context API
date: 2019-09-04 05:00:00
categories:
- GoLang
tags:
- 开发
description: 详解golang Context API如何使用
---

# 介绍

以一个简单的问题开始. 有一个每秒进行一些处理动作的函数：

```go
func Perform() {
    for {
        SomeFunction()
        time.Sleep(time.Second)
    }
}
```

我们这样运行它：

```go
go Perform()
```

有个想法是在有明确的取消动作或者超过截至时间时，取消这个函数。Context就是为了这个目的而设计的，让我们看一下context.Context接口：

```go
type Context interface {
    Done() <-chan struct{}
    Err() error
    Deadline() (deadline time.Time, ok bool)
    Value(key interface{}) interface{}
}
```

分析一下：

* ctx.Done()  返回一个取消信号的channel，用于判断这个context是不是被取消了。
* ctx.Err() 返回取消的原因（超过截至时间，或者是取消信号）。
* ctx.Deadline() 返回截至时间，如果被设置的话。
* ctx.Value(key) 返回对应key的值。

那么有几个问题：

* 为什么 ctx.Done()返回一个channel？为什么不返回一个bool类型的值？
* 为什么没有cancel方法？
* 怎样设置deadline？
* ctx.Value(key)是干什么的？

为了理解这些API，了解它被设计成满足以下两个要求的原因是非常有用的：

1. **取消应该是建议性的**

&emsp;&emsp;每个函数自行return是它自己的工作，且调用者不知道被调用函数的内部情况，因此不应该强行终止被调用函数的执行，而应该通知被调用函数不再需要它的工作了。
&emsp;&emsp;调用者发送取消信息，让被调用函数决定怎么进一步处理。例如，当函数获知它的工作是不再被需要的时候，函数可以自清理并提前返回 。

2. **取消信号应该是可传递的**

&emsp;&emsp;当取消一个函数, 我们也需要取消它调用的所有函数。这意味着取消信号应该被广播，从调用者到它各个层级的子函数。

# 创建一个context

一个简单的方法，用context.Background()创建一个context：

```go
ctx := context.Background()
```

context.Background() 返回一个空的context。由于取消是建议性和可传递的，我们应该给每个函数一个代表取消信息的参数。

我们修改一下程序：

```go
go Perform()
```

变成

```go
ctx := context.Background()
go Perform(ctx)
```

# 设置deadline

空context是没用的。我们需要设置deadline或者能够手动取消它。然而，context.Context 接口只定义了查询方法。我们无法修改deadline。
我们不能修改context的原因是：我们希望阻止带context参数的函数去修改或者取消一个请求。
context中信息流的方向是严格的从父函数流向子函数的。例如，当用户关闭浏览器中的一个标签（代表父）， 隶属这个标签的所有函数（代表子）都应该被关闭。

因此，我们得到一个被设置了截止日期的上下文：

```go
ctx, cancel := context.WithDeadline(parentContext, time)
//或者
ctx, cancel := context.WithTimeout(parentContext, duration)
```

我们可以使用cancel()来提醒Perform函数，我们不再需要它的工作了。下面，我们将看Perform如何处理此信号。

# 检查context是否被取消

取消事件应该向下广播到所有被调用的函数。Go中channel有一个特性，使它很适合这个目的：从一个已被关闭的channel接收，会立即返回零值。
这意味着多个函数可以监听同一个channel，当它关闭时，所有函数都能接收到取消信号。
Done方法返回在一个只读通道（cancel函数的功能就是关闭这个channel）。这有一个检查context是否被取消的简单示例：

```go
func Perform(ctx context.Context) {
    for {
        SomeFunction()
 
        select {
        case <-ctx.Done():
            // ctx is canceled
            return
        default:
            // ctx is not canceled, continue immediately
        }
    }
}
```

请注意，select语句不会阻塞，因为它有一个default语句。这会导致for循环立即执行SomeFunction。我们需要在每次迭代之间休眠1秒钟：

```go
func Perform(ctx context.Context) {
    for {
        SomeFunction()
 
        select {
        case <-ctx.Done():
            // ctx is canceled
            return
        case <-time.After(time.Second):
            // wait for 1 second
        }
    }
}
```

当context被取消时，我们可以通过调用ctx.Err()找出原因。

```go
func Perform(ctx context.Context) error {
    for {
        SomeFunction()
 
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(time.Second):
            // wait for 1 second
        }
    }
    return nil
}
```

ctx.Err()只有两个可能的值：context.DeadlineExceeded和context.Canceled。ctx.Err()只有在 ctx.Done()被关闭后才会被调用。ctx被取消之前的ctx.Err()的结果是未被设置的。
如果SomeFunction会执行很长时间，我们也可以让它知道取消信号。我们通过将ctx作为其一个参数传递给它来实现这一点。

```go
func Perform(ctx context.Context) error {
    for {
        SomeFunction(ctx)
 
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(time.Second):
            // wait for 1 second
        }
    }
    return nil
}
```

# context.TODO()是什么?

和context.Background相似，创建context的另一个方法是：

```go
ctx := context.TODO()
```

TODO函数也返回一个空的context。TODO被用于重构函数去支持context。当在函数中无法获取一个父context时，我们使用它 . 所有的TODO context最终都应该被其他的context替换掉。

# ctx.WithValue是什么?

context的最常见用法是处理请求中的取消。为实现这一点，context通常贯穿请求的整个生命周期（例如，作为所有函数的第一个参数）。
贯穿请求的生命周期中的另一个有用信息是数据的值：诸如用户会话和登录信息之类。context包也可以轻松地将这些值存储在context实例中。因为它们与取消信息共享相同的调用路径。
要设置数据的值，我们使用context.WithValue函数派生一个context：

```go
ctx := context.WithValue(parentContext, key, value)
```

要从ctx或从ctx派生的任何context中检索此值，请使用：

```go
value := ctx.Value(key)
```

# 相关资料

[Pipelines and cancellation](https://blog.golang.org/pipelines)


