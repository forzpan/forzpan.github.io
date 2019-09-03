---
title: golang 易错点
date: 2019-09-03 10:00:00
categories:
- GoLang
tags:
- 开发
description: golang开发中遇到的几个易错点
---

* 外层有`a，b`时，用`a,b,c：=`会创建所有变量，而不仅仅是新出现的`c`。

```go
func fn() {
    var a, b int
    {
        a, b, c := 1, 1, 1
        fmt.Println(a, b, c)
    }
    fmt.Println(a, b)
}
```

* 使用defer时，一定要用`func xx()(err error)`明示返回变量的名字,这样内层`return err`会进行一次赋值，不然内存`retrun`后，`defer`读的是外层`err`，是空的。

```go
func fn() error {
    //初始化一个error
    var err error
    //注册defer处理
    defer func() {
        //判断err（是外层err）
        if err != nil {
            fmt.Println("haha")
        }
    }()
    //某种场景提前返回
    if 1 == 1 {
        //这种场景触发某种err，这边可能为了方便使用了 := 重建了局部err
        val, err := 1, errors.New("text string")
        _ = val
        //这边返回的是局部err，defer里无法捕获这个err
        return err
    }
    //返回全局err
    return err
}
```

修改一下

```go
func fn() (err error) {
    //注册defer处理
    defer func() {
        //判断err（是外层err）
        if err != nil {
            fmt.Println("haha")
        }
    }()
    //某种场景提前返回
    if 1 == 1 {
        //这种场景触发某种err，这边可能为了方便使用了 := 重建了局部err
        val, err := 1, errors.New("text string")
        _ = val
        //这边返回的是局部err，但是实际上还有个隐式的赋值，局部err赋值给全局err，然后defer就捕获的err就非nil了
        return err
    }
    //返回全局err
    return
}
```

* `slice`，`string`等是本质是结构体，`map`,`channel`本质是结构体指针。go中只有值传递。

[说说不知道的Golang中参数传递](https://gocn.vip/article/1529)
[There is no pass-by-reference in Go](https://dave.cheney.net/2017/04/29/there-is-no-pass-by-reference-in-go)

```go
func fn(m map[int]int) {
        m = make(map[int]int)
}
 
func main() {
        var m map[int]int
        fn(m)
        fmt.Println(m == nil)
}
```


