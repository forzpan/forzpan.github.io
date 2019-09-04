---
title: golang 不得不知道的知识点之nil
date: 2019-09-04 13:00:00
categories:
- GoLang
tags:
- 开发
description: 深入分析golang中的nil
---

# 前言

golang中的nil，很多人都误以为与Java、PHP等编程语言中的null一样。但是实际上Golang的nil复杂得多了。

nil 为预声明的标示符，定义在builtin/builtin.go：

```go
// nil is a predeclared identifier representing the zero value for a pointer, channel, func, interface, map, or slice type.

// Type must be a pointer, channel, func, interface, map, or slice type.

var nil Type

// Type is here for the purposes of documentation only. It is a stand-in for any Go type, but represents the same type for any given function invocation.

type Type int
```

# 零值

按照Go语言规范，任何类型在未初始化时都对应一个零值：布尔类型是false，整型是0，字符串是""，而指针、函数、interface、slice、channel和map的零值都是nil。

**PS：这里没有说结构体struct的零值为nil，因为struct的零值与其属性有关**

nil没有默认的类型，尽管它是多个类型的零值，必须显式或隐式指定每个nil的明确类型。

```go
package main
 
func main() {
    // 明确.
    _ = (*struct{})(nil)
    _ = []int(nil)
    _ = map[int]bool(nil)
    _ = chan string(nil)
    _ = (func())(nil)
    _ = interface{}(nil)
    // 隐式.
    var _ *struct{} = nil
    var _ []int = nil
    var _ map[int]bool = nil
    var _ chan string = nil
    var _ func() = nil
    var _ interface{} = nil
}
```

如果关注过golang关键字的同学就会发现，里面并没有nil，也就是说nil并不是关键字，那么就可以在代码中定义nil，那么nil就会被隐藏。

```go
package main
 
import "fmt"
 
func main() {
    nil := 123
    fmt.Println(nil) // 123
    var _ map[string]int = nil //cannot use nil (type int) as type map[string]int in assignment
}
```

# nil的地址和值大小

不同类型的nil的内存地址是一样的。

```go
package main
 
import (
    "fmt"
)
 
func main() {
    var m map[int]string
    var ptr *int
    var sl []int
    fmt.Printf("%p\n", m)   //0x0
    fmt.Printf("%p\n", ptr) //0x0
    fmt.Printf("%p\n", sl)  //0x0
}
```

nil值的大小始终与其类型的non-nil值大小相同。因此, 表示不同类型的零值nil可能具有不同的大小。

```go
package main
 
import (
    "fmt"
    "unsafe"
)
 
func main() {
    var p *struct{} = nil
    fmt.Println(unsafe.Sizeof(p)) // 8
 
    var s []int = nil
    fmt.Println(unsafe.Sizeof(s)) // 24
 
    var m map[int]bool = nil
    fmt.Println(unsafe.Sizeof(m)) // 8
 
    var c chan string = nil
    fmt.Println(unsafe.Sizeof(c)) // 8
 
    var f func() = nil
    fmt.Println(unsafe.Sizeof(f)) // 8
 
    var i interface{} = nil
    fmt.Println(unsafe.Sizeof(i)) // 16
}
```

大小是编译器和体系结构所依赖的。以上打印结果为64位体系结构和正式 Go 编译器。对于32位体系结构, 打印的大小将是一半。

对于正式 Go 编译器, 同一种类的不同类型的两个nil值的大小始终相同。例如, 两个不同的切片类型 ( []int和[]string) 的两个nil值始终相同。

# nil值比较

> **(1)不同类型的nil是不能比较的**

```go
package main
 
import (
    "fmt"
)
 
func main() {
    var m map[int]string
    var ptr *int
    fmt.Printf(m == ptr) //invalid operation: m == ptr (mismatched types map[int]string and *int)
}
```

在 Go 中, 两个不同可比较类型的两个值只能在一个值可以隐式转换为另一种类型的情况下进行比较。

具体来说, 有两个案例两个不同的值可以比较：

* 两个值之一的类型是另一个的基础类型
* 两个值之一的类型实现了另一个值的类型 (必须是接口类型)

nil值比较并没有脱离上述规则。

```go
package main
 
import (
    "fmt"
)
 
func main() {
    type IntPtr *int
    fmt.Println(IntPtr(nil) == (*int)(nil))        //true
    fmt.Println((interface{})(nil) == (*int)(nil)) //false
}
```

> **(2)同一类型的两个nil值可能无法比较**

因为Go中存在map、slice和函数类型是不可比较类型，它们有一个别称为不可比拟的类型，所以比较它们的nil亦是非法的。

```go
package main
 
import (
    "fmt"
)
 
func main() {
    var v1 []int = nil
    var v2 []int = nil
    fmt.Println(v1 == v2)
    fmt.Println((map[string]int)(nil) == (map[string]int)(nil))
    fmt.Println((func())(nil) == (func())(nil))
}
```

不可比拟的类型的值是可以与“纯nil”进行比较的：

```go
package main
 
import (
    "fmt"
)
 
func main() {
    fmt.Println((map[string]int)(nil) == nil) //true
    fmt.Println((func())(nil) == nil)         //true
}
```

> **(3)两nil值可能不相等**

如果两个比较的nil值之一是一个接口值, 而另一个不是, 假设它们是可比较的, 则比较结果总是 false。

原因是在进行比较之前, 接口值将转换为接口值的类型。转换后的接口值具有具体的动态类型, 但其他接口值没有。这就是为什么比较结果总是false的。

```go
package main
 
import (
    "fmt"
)
 
func main() {
    fmt.Println((interface{})(nil) == (*int)(nil)) // false
}
```

# 其他

map的key可以为指针、函数、interface、slice、channel和map类型，所以key可以为nil。

```go
package main
 
import (
    "fmt"
)
 
func main() {
    mmap := make(map[*string]int, 4)
    a := "a"
    mmap[&a] = 1
    mmap[nil] = 99
    fmt.Println(mmap) //map[0xc042008220:1 <nil>:99]
}
```

相关链接：[不得不知道的golang知识点之nil](https://gocn.vip/article/478)


