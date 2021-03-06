---
title: golang 关闭channel的正确姿势
date: 2019-09-04 07:00:00
categories:
- GoLang
tags:
- 开发
description: golang中关闭channel的注意点
---

channel是什么不多赘言。go关键字+channel，写并发那叫一个爽。

但经常有些“痛点”被提及：

* 关闭一个已经被关闭的channel会导致panic。
* 像一个已经关闭的channel发送数据会导致panic。（特别是多生产者的状况下）

这个时候心里就吐槽，为什么没有提供一个函数比如 IsClosed() 来判断一个channel是否被关闭了呢？

立刻设计一个：

```go
func IsClosed(ch <-chan T) bool {
    select {
    case _, ok := <-ch:
        return !ok
    default:
    }
    return false
}
```

看似很美好，实际用处有限

```go
if !IsClosed(ch) {
　　close(ch)
}
```

并发场景下，判断时候的状态和close的状态可并不一定是相同的。重复close还是可能发生。

所以，以上是相当然的用法，还有其他，比如recover，sync.Once，sync.Mutex等虽然勉强可用，但是还是属于野路子。

所以最好遵守下面几条规则：

* 谁创建的channel谁负责关闭
* 不要关闭有发送者的channel
* 作为函数参数的channel最好带方向

第一点，哪个go程（包括主程）创建了channel，就有义务去维护它，从生到死。我们定义为这个go程为这个channel的宿主好了。

第二点，channel的宿主，需要保证关闭channel之前，这个channel的上游go程不再发送。进一步说，宿主能够通知上游go程，让它停止发送，并且被反馈已停止，这个通知和反馈一般也是可以由另外的channel实现。

第三点，作为函数的参数，channel带方向是有一定的好处的。可以明确一个channel的使用方式。另外对于一个 <-chan ，函数内部是没有权限关闭它的，所以不会发生从接收端关闭channel的情况。

（对于需要无方向channel作为参数的函数，比如一个用作控制最大并发数量的带缓存channel，处理之前从中取一个，处理完成后还回去一个。拆成两个带方向的channel参数也没什么不好，虽然其实是同一个）。

所以说，可以这样分。从最简单的   M生产者--->channel--->N消费者 模型中。

从上帝视角，有个主程序，它建了一堆的channel，作用上分两种   数据channel  和  消息channel。

数据channel实际就一个，作为生产者和消费者之间传递数据的通道，就如同一个流水线，一般还是带缓存的。

消息channel，用于主程序和各个go程之间的传递各种信号，让各个go程有所判断，进行不同的动作。

比如想停止整个体系，可以先发送操作系统信号给主程序，主程序捕获后，广播停止信号给所有生产者，生产获悉后，停止发送，然后反馈给主程序，

主程序获悉所有生产者停止后，关闭数据管道，消费者获悉消费完管道内所有数据，反馈给主程序退出信号。主程序退出。

有时候，一些 消息channel，设计成带1个缓存比较好。

可以继续丰富功能，主要着手是消息channel，发送定时任务；主程序根据消费能力，自适应生产能力；增加多channel等等。

另外还可以支持多数据channel，数据channel中数据的持久化，预处理等等。

相关连接 [如何优雅地关闭Go channel](https://www.jianshu.com/p/d24dfbb33781)



