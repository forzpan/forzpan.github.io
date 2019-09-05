---
title: golang flag包详解
date: 2019-09-05 02:00:00
categories:
- GoLang
tags:
- 开发
description: 详细说明golang中flag包使用
---

# 从一个例子开始

在写命令行程序（工具、server）时，对命令参数进行解析是常见的需求。举个例子：

```
nginx version: nginx/1.10.0
Usage: nginx [-?hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]
 
Options:
  -?,-h         : this help
  -v            : show version and exit
  -V            : show version and configure options then exit
  -t            : test configuration and exit
  -T            : test configuration, dump it and exit
  -q            : suppress non-error messages during configuration testing
  -s signal     : send signal to a master process: stop, quit, reopen, reload
  -p prefix     : set prefix path (default: /usr/local/nginx/)
  -c filename   : set configuration file (default: conf/nginx.conf)
  -g directives : set global directives out of configuration file
```

各种语言一般都会提供解析命令行参数的方法或库，以方便程序员使用。如果命令行参数纯粹自己写代码解析，对于比较复杂的，还是挺费劲的。

在 go标准库 中提供了一个包：flag，方便进行命令行解析。我们通过 flag 实现类似 nginx 的这个输出，创建文件 nginx.go，内容如下：

```go
package main
 
import (
    "flag"
    "fmt"
    "os"
)
 
// 实际中应该用更好的变量名
var (
    h bool
 
    v, V bool
    t, T bool
    q    *bool
 
    s string
    p string
    c string
    g string
)
 
func init() {
    flag.BoolVar(&h, "h", false, "this help")
    flag.BoolVar(&v, "v", false, "show version and exit")
    flag.BoolVar(&V, "V", false, "show version and configure options then exit")
    flag.BoolVar(&t, "t", false, "test configuration and exit")
    flag.BoolVar(&T, "T", false, "test configuration, dump it and exit")
 
    // 另一种绑定方式
    q = flag.Bool("q", false, "suppress non-error messages during configuration testing")
 
    // 注意 `signal`。默认是 -s string，有了 `signal` 之后，变为 -s signal
    flag.StringVar(&s, "s", "", "send `signal` to a master process: stop, quit, reopen, reload")
    flag.StringVar(&p, "p", "/usr/local/nginx/", "set `prefix` path")
    flag.StringVar(&c, "c", "conf/nginx.conf", "set configuration `file`")
    flag.StringVar(&g, "g", "conf/nginx.conf", "set global `directives` out of configuration file")
 
    // 改变默认的 Usage
    flag.Usage = usage
}
 
func main() {
    flag.Parse()
 
    if h {
        flag.Usage()
    }
}
 
func usage() {
    fmt.Fprintf(os.Stderr, `nginx version: nginx/1.10.0
Usage: nginx [-hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]
 
Options:
`)
    flag.PrintDefaults()
}
```

执行：go run nginx.go -h，（或 go build -o nginx && ./nginx -h）输出如下：

```
nginx version: nginx/1.10.0
Usage: nginx [-hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]
 
Options:
  -T    test configuration, dump it and exit
  -V    show version and configure options then exit
  -c    file
        set configuration file (default "conf/nginx.conf")
  -g    directives
        set global directives out of configuration file (default "conf/nginx.conf")
  -h    this help
  -p    prefix
        set prefix path (default "/usr/local/nginx/")
  -q    suppress non-error messages during configuration testing
  -s    signal
        send signal to a master process: stop, quit, reopen, reload
  -t    test configuration and exit
  -v    show version and exit
```

# 定义命令行参数有两种方式

* flag.Xxx()，其中 Xxx 可以是 Int、String 等；返回一个相应类型的指针，如：

```go
var ip = flag.Int("flagname", 1234, "help message for flagname")
```

* flag.XxxVar()，将 flag 绑定到一个变量上，如：

```go
var flagvar int
flag.IntVar(&flagvar, "flagname", 1234, "help message for flagname")
```

# 自定义 Value

创建自定义某种类型的 flag，只要实现 flag.Value 接口即可（要求 receiver 是指针），这时候可以通过如下方式定义该 flag：

```go
flag.Var(&flagVal, "name", "help message for flagname")
```

例如，解析我喜欢的编程语言，我们希望直接解析到 slice 中，我们可以定义如下 Value：

```go
type sliceValue []string
 
func newSliceValue(vals []string, p *[]string) *sliceValue {
    *p = vals
    return (*sliceValue)(p)
}
 
func (s *sliceValue) Set(val string) error {
    *s = sliceValue(strings.Split(val, ","))
    return nil
}
 
func (s *sliceValue) Get() interface{} { return []string(*s) }
 
func (s *sliceValue) String() string { return strings.Join([]string(*s), ",") }
```

之后可以这么使用：

```go
var languages []string
flag.Var(newSliceValue([]string{}, &languages), "slice", "I like programming `languages`")
```

这样通过 -slice "go,php" 这样的形式传递参数，languages 得到的就是 [go, php]。

flag 中对 Duration 这种非基本类型的支持，使用的就是类似这样的方式。

# 解析 flag

在所有的 flag 定义完成之后，可以通过调用 flag.Parse() 进行解析。

命令行 使用 flag 的语法有如下三种形式：

* -flag&emsp;&emsp;// 只支持bool类型
* -flag x &emsp;// 只支持非bool类型
* -flag=x

其中第二种形式只能用于非bool类型的flag，因为：如果也支持bool类型，那么对于cmd -debug filename这样的命令，如果文件名字是：0或false，则命令的原意被改变。

如果bool类型不支持-flag这种形式，则bool类型可以和其他类型一样处理。也正因为bool类型不支持第二种形式，Parse()中，对 bool 类型进行了特殊处理。

默认的，提供了-flag，则对应的值为 true，否则为 flag.Bool/BoolVar 中指定的默认值。如果希望显示设置为 false 则使用 -flag=false。

> int 类型可以是十进制、十六进制、八进制甚至是负数。
> bool 类型可以是1, 0, t, f, true, false, TRUE, FALSE, True, False。
> Duration类型可以接受任何time.ParseDuration能解析的类型。

# 包内部概览

ErrHelp：该错误类型用于当命令行指定了 ·-help` 参数但没有定义时。

Usage：这是一个函数，用于输出所有定义了的命令行参数和帮助信息。一般当命令行参数解析出错时，该函数会被调用。我们可以自定义Usage。

go标准库中，经常这么做：

> 定义了一个类型，提供了很多方法；
> 为了方便使用，会实例化一个该类型的通用实例，这样便可以直接使用该实例调用方法。
> 比如：encoding/base64 中提供了StdEncoding和URLEncoding实例，使用时：base64.StdEncoding.Encode()。

flag包也使用了类似的方法，比如CommandLine实例，只不过flag进行了进一步封装：

将FlagSet的方法都重新定义了一遍，也就是提供了一系列函数，而函数中只是简单的调用已经实例化好了的FlagSet实例：CommandLine的方法。

这样，使用者可以直接调用：flag.Parse() 而不是 flag.CommandLine.Parse()。

## ErrorHandling

```go
type ErrorHandling int
```

该类型定义了在参数解析出错时错误处理方式。定义了三个该类型的常量：

```go
const (
    ContinueOnError ErrorHandling = iota
    ExitOnError
    PanicOnError
)
```

三个常量在源码的 FlagSet 的方法 parseOne() 中使用了。

## Flag

```go
// A Flag represents the state of a flag.
type Flag struct {
    Name     string // name as it appears on command line
    Usage    string // help message
    Value    Value  // value as set
    DefValue string // default value (as text); for usage message
}
```

Flag 类型代表一个 flag 的状态。

比如，对于命令：./nginx -c /etc/nginx.conf，相应代码是：

```go
flag.StringVar(&c, "c", "conf/nginx.conf", "set configuration `file`")
```

则该 Flag 实例（可以通过 flag.Lookup("c") 获得）相应各个字段的值为：

```go
&Flag{
    Name: c,
    Usage: set configuration file,
    Value: /etc/nginx.conf,
    DefValue: conf/nginx.conf,
}
```

## FlagSet

```go
// A FlagSet represents a set of defined flags.
type FlagSet struct {
    // Usage is the function called when an error occurs while parsing flags.
    // The field is a function (not a method) that may be changed to point to
    // a custom error handler.
    Usage func()
 
    name string // FlagSet的名字。CommandLine 给的是 os.Args[0]
    parsed bool // 是否执行过Parse()
    actual map[string]*Flag // 存放实际传递了的参数（即命令行参数）
    formal map[string]*Flag // 存放所有已定义命令行参数
    args []string // arguments after flags // 开始存放所有参数，最后保留 非flag（non-flag）参数
    exitOnError bool // does the program exit if there's an error?
    errorHandling ErrorHandling // 当解析出错时，处理错误的方式
    output io.Writer // nil means stderr; use out() accessor
}
```

## Value接口

```go
// Value is the interface to the dynamic value stored in a flag.
// (The default value is represented as a string.)
type Value interface {
    String() string
    Set(string) error
}
```
所有参数类型需要实现 Value 接口，flag 包中，为int、float、bool等实现了该接口。借助该接口，我们可以自定义flag。（上文已经给了具体的例子）

# 主要方法

## 实例化方式

NewFlagSet() 用于实例化 FlagSet。预定义的 FlagSet 实例 CommandLine 的定义方式：

```go
// The default set of command-line flags, parsed from os.Args.
var CommandLine = NewFlagSet(os.Args[0], ExitOnError)
```

可见，默认的 FlagSet 实例在解析出错时会退出程序。

由于FlagSet中的字段是非导出的，其他方式获得FlagSet实例后，比如：FlagSet{} 或 new(FlagSet)，应该调用Init()方法初始化name和errorHandling，否则name为空，errorHandling为ContinueOnError。

## 解析参数（Parse）

```go
func (f *FlagSet) Parse(arguments []string) error
```

参数列表中解析定义的flag。方法参数arguments不包括命令名，即应该是os.Args[1:]。事实上，flag.Parse()函数就是这么做的：

```go
// Parse parses the command-line flags from os.Args[1:].  Must be called
// after all flags are defined and before flags are accessed by the program.
func Parse() {
    // Ignore errors; CommandLine is set for ExitOnError.
    CommandLine.Parse(os.Args[1:])
}
```

该方法应该在 flag 参数定义后而具体参数值被访问前调用。

如果提供了 -help 参数（命令中给了）但没有定义（代码中没有），该方法返回 ErrHelp 错误。默认的 CommandLine，在 Parse 出错时会退出程序（ExitOnError）。

为了更深入的理解，我们看一下 Parse(arguments []string) 的源码：

```go
func (f *FlagSet) Parse(arguments []string) error {
    f.parsed = true
    f.args = arguments
    for {
        seen, err := f.parseOne()
        if seen {
            continue
        }
        if err == nil {
            break
        }
        switch f.errorHandling {
        case ContinueOnError:
            return err
        case ExitOnError:
            os.Exit(2)
        case PanicOnError:
            panic(err)
        }
    }
    return nil
}
```

真正解析参数的方法是非导出方法 parseOne。

结合 parseOne 方法，我们来解释 non-flag 以及包文档中的这句话：

> Flag parsing stops just before the first non-flag argument ("-" is a non-flag argument) or after the terminator "--".

我们需要了解解析什么时候停止。

根据Parse()中for循环终止的条件（不考虑解析出错），我们知道，当parseOne返回false, nil时，Parse解析终止。正常解析完成我们不考虑。

看一下parseOne的源码发现，有两处会返回false, nil：

* 第一个 non-flag 参数:

```go
s := f.args[0]
if len(s) == 0 || s[0] != '-' || len(s) == 1 {
    return false, nil
}
```

也就是，当遇到 单独一个"-" 或 不是"-"开头的参数 时，会停止解析。比如：

```
./nginx  -  -c 或 ./nginx  build  -c
```

这两种情况，-c 都不会被正确解析。像该例子中的"-"或build（以及之后的参数），我们称之为 non-flag 参数。


* 两个连续的"--"

```go
if s[1] == '-' {
    num_minuses++
    if len(s) == 2 { // "--" terminates the flags
        f.args = f.args[1:]
        return false, nil
    }
}
```

也就是，当遇到连续的两个"-"时，解析停止。

说明：这里说的"-"和"--"，位置和"-c"这种的一样。也就是说，不包括下面这种情况：

```
./nginx -c --
```

这里的"--"会被当成是 c 的值。

parseOne方法中接下来是处理-flag=x这种形式，然后是-flag这种形式（bool类型）（这里对bool进行了特殊处理），接着是-flag x这种形式，最后将解析成功的Flag实例存入FlagSet的actual map中。

另外，在 parseOne 中有这么一句：

```go
f.args = f.args[1:]
```

也就是说，每执行成功一次 parseOne，f.args 会少一个。所以，FlagSet 中的 args 最后留下来的就是所有 non-flag 参数。


## Arg(i int) 和 Args()、NArg()、NFlag()

Arg(i int) 和 Args() 这两个方法就是获取 non-flag 参数的。

## NArg()获得 non-flag 的个数。

NFlag() 获得 FlagSet 中 actual 长度（即被设置了的参数个数）。

## Visit/VisitAll

这两个函数分别用于访问 FlatSet 的 actual 和 formal 中的 Flag，而具体的访问方式由调用者决定。

## PrintDefaults()

打印所有已定义参数的默认值（调用 VisitAll 实现），默认输出到标准错误，除非指定了 FlagSet 的 output（通过SetOutput() 设置）

## Set(name, value string)

设置某个 flag 的值（通过 name 查找到对应的 Flag）。

# 其他

如项目需要复杂或更高级的命令行解析方式，可以使用[https://github.com/urfave/cli](https://github.com/urfave/cli)或[https://github.com/spf13/cobra](https://github.com/spf13/cobra)这两个强大的库。

[原文连接](https://books.studygolang.com/The-Golang-Standard-Library-by-Example/chapter13/13.1.html)


