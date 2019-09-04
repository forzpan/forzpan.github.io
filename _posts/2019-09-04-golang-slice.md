---
title: golang [N]byte、string和[]byte
date: 2019-09-04 08:00:00
categories:
- GoLang
tags:
- 开发
description: golang中数组，字符串，切片的使用与转换
---

# [N]byte

字节数组，占用一块定长连续的内存，可读写。非0大小数组的长度不得超过2GB

# string

字串本质是逻辑上只读的字节数组，零值是""而不是 nil。src/runtime/string.go 中字串结构定义如下：

```go
type stringStruct struct {
    str unsafe.Pointer　　//一个指针
    len int　　　　　　　　//固定长度
}
```

零值""对应的底层结构体的值是： 指针是nil，len是0；有几个注意点：

* string常量会在编译期分配到只读段，对应数据地址不可写入。
* 相同的string常量不会重复存储，但动态生成（比如fmt.Sprint等编译器无法优化确定结果的方式）的字符串即使内容一样，数据也是在不同的空间。
* 只有动态生成的string可以用unsafe魔改（动态生产的string并不存在只读段，因此通过将string底层数组作为一个数组来处理的时候，是可写的）。
* string和[]byte转换，会将数据复制到堆上，返回数据指向复制的数据。所以string(bytes)存在开销。

```go
func main() {
    str1 := ""
    str2 := "1234"
    str3 := str1
    str4 := "12" + "" + "34"
    str5 := fmt.Sprint("1234")
    str6 := strings.Join([]string{"12", "34"}, "")
    str1str := *(*stringStruct)(unsafe.Pointer(&str1))
    str2str := *(*stringStruct)(unsafe.Pointer(&str2))
    str3str := *(*stringStruct)(unsafe.Pointer(&str3))
    str4str := *(*stringStruct)(unsafe.Pointer(&str4))
    str5str := *(*stringStruct)(unsafe.Pointer(&str5))
    str6str := *(*stringStruct)(unsafe.Pointer(&str6))
    fmt.Println(str1str, str2str, str3str, str4str, str5str, str6str)
}
```

# []byte

[]byte的底层数据结构，src/runtime/slice.go中定义如下：

```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

和string不一样的地方是，多了个cap，最大可用的长度就是说len的可变是有限制的，实际也算个空间换时间呗。

# 转换

> [N]byte 和 string，把string理解成[N]byte的只读版好了，但是两个不能直接转换可以用for循环一个一个byte复制，或者用指针魔法：

```go
func main() {
    str1 := "1234"
    arr1 := *(*[4]byte)((*stringStruct)(unsafe.Pointer(&str1)).str)
    fmt.Println(arr1)
}
```

[N]byte转string魔法  就是把string看成stringStruct，初始化一下就行。

> [N]byte和[]byte转

可以只有slice格式直接取出其中一部分，也可以用指针魔法，这和string类似。

> string 和 []byte

这两可以直接强制类型转换。两个底层结构体很像，然后一个是只读，一个是可写，转换涉及到内存copy，有些开销。

```go
func main() {
    arr := []byte{'1', '2', '3', '4'}
    str2 := string(arr)
 
    str := fmt.Sprint("1234")
    arr2 := []byte(str)
 
    arrp := *(*unsafe.Pointer)(unsafe.Pointer(&arr))
    str2p := *(*unsafe.Pointer)(unsafe.Pointer(&str2))
    strp := *(*unsafe.Pointer)(unsafe.Pointer(&str))
    arr2p := *(*unsafe.Pointer)(unsafe.Pointer(&arr2))
 
    fmt.Println(arrp, str2p, strp, arr2p)
}
```

原地转换黑魔法

```go
func stringtoslicebyte(s string) []byte {
    sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
    bh := reflect.SliceHeader{
        Data: sh.Data,
        Len:  sh.Len,
        Cap:  sh.Len,
    }
    return *(*[]byte)(unsafe.Pointer(&bh))
}
 
func slicebytetostring(b []byte) string {
    bh := (*reflect.SliceHeader)(unsafe.Pointer(&b))
    sh := reflect.StringHeader{
        Data: bh.Data,
        Len:  bh.Len,
    }
    return *(*string)(unsafe.Pointer(&sh))
}
```

相关链接 [golang string和[]byte的对比](https://gocn.vip/article/467)


