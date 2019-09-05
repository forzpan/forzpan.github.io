---
title: golang Template使用
date: 2019-09-05 01:00:00
categories:
- GoLang
tags:
- 开发
description: 详细说明golang中Template的使用
---

{% raw %}

GO Template 通过 `{{}}` 来包含在渲染时被替换的字段

`{{.}}` 表示struct或者non-struct值

`{{.FieldName}}` 访问struct的字段（访问结构体内部的非导出字段，Execute跑到那个位置时会报错，比如`{{.fieldName}}`。不存在的字段当然也报错）

{% endraw %}

# if语句

```
{% raw %}
{{if .FieldName}}
    Value of FieldName is {{ .FieldName }}
{{ else }}
    Value of FieldName is null
{{end}}
{% endraw %}
```

# range语句

```
{{range .Members}} -- Members可以是slice,array,map等
    {{.}} -- 代表一个Member元素
    {{.Name}} -- 代表这个Member元素的Name，比如这个Member也是个struct。
{{end}}
```

# 变量

with,range,if 都可以声明变量

```
{{with $x := "output"}}
    {{printf "%q" $x}}  -- with-end区域内这个变量有效
{{end}}
```

# 函数

* 内置函数
    * `html` 返回html转换过的内容
    * `js` &emsp;返回js转换过的内容

+ 自定义函数，例如：

```go
func EmailDealWith(args …interface{}) string
t = t.Funcs(template.FuncMap{"emailDeal": EmailDealWith})
```

就可以，如下方法使用

```
{{range .Emails}}
    an emails {{.|emailDeal}}
{{end}}
```

# 管道

```
{{. | html}}
{{with $x := "output" | printf "%q"}}{{$x}}{{end}}
{{with $x := "output"}}{{$x | printf "%q"}}{{end}}
```

# 管道

```
{{. | html}}
{{with $x := "output" | printf "%q"}}{{$x}}{{end}}
{{with $x := "output"}}{{$x | printf "%q"}}{{end}}
```

# 嵌套模版

定义：

```
{{define "子模板名称"}}
    内容
{{end}}
```

使用：

```
{{template "子模板名称"}}
```

例子：

```
//header.tmpl
{{define "header"}}
<html>
<head>
    <title>演示信息</title>
</head>
<body>
{{end}}
 
//content.tmpl
{{define "content"}}
{{template "header"}}
<h1>演示嵌套</h1>
<ul>
    <li>嵌套使用define定义子模板</li>
    <li>调用使用template</li>
</ul>
{{template "footer"}}
{{end}}
 
//footer.tmpl
{{define "footer"}}
</body>
</html>
{{end}}
```

# template包

`text/template` 和 `html/template` 使用方法基本相同，`html/template` 内部也使用了 `text/template` 包。

主要有三个方法：

* `New` 创建模版（底层数据结构）
* `Parse` 解析模版（装载模版内容）
* `Execute` 应用模版（替换模版里的变量，并执行模版里的逻辑，最终输出结果）

+ 例子1：

```go
package main
 
import (
    "os"
    "text/template"
)
 
type Member struct {
    Name        string
    Description string
}
 
func main() {
    member := Member{"huluwa", "dawa erwa sanwa..."}
    t := template.New("member")
    t, err := t.Parse("Member named {{ .Name}} with description: {{ .Description}}")
    if err != nil {
        panic(err)
    }
    err = t.Execute(os.Stdout, member)
    if err != nil {
        panic(err)
    }
}
```

+ 例子2：

```go
package main
 
import (
    "html/template"
    "os"
)
 
type entry struct {
    Name string
    Done bool
}
 
type ToDo struct {
    User string
    List []entry
}
 
func main() {
    todo := ToDo{
        User: "xiaogang",
        List: []entry{ {"shuxue", true}, {"yuwen", false}, {"dili", true} },
    }
 
    t := template.New("xx.tmpl")
    t, err := t.ParseFiles("E:\\vscode-go\\project\\src\\just\\xx.tmpl")
    if err != nil {
        panic(err)
    }
 
    err = t.Execute(os.Stdout, todo)
    if err != nil {
        panic(err)
    }
}
```

例子2中`t.Execute`会寻找很模版文件名相同的模板，也就是`xx.tmpl`命名的模版。

如果`template.New`命名不是`xx.tmpl`，就会报错`panic: template: "xx" is an incomplete or empty template`。

原因是，`template.New`会产生一个template，实际上ParseFiles时，如果文件名字命名的template不存在，会创建一个文件名字命名的template，存储对应的信息。

所以`template.New`可能并没有什么鸟用。

另外`t.ExecuteTemplate`也可以使用，直接指定真实使用的模版。内部`{{define "模板名称"}}内容{{end}}`也会创建子模板。

`template.Must`也没什么好讲的就是一层封装，返回一个值，可以方便把函数一个接一个的串起来使用。

+ 例子3

```go
package main
 
import (
    "fmt"
    "html/template"
    "os"
    "strings"
)
 
type Friend struct {
    Fname string
}
 
type Person struct {
    UserName string
    Emails   []string
    Friends  []*Friend
}
 
func EmailDealWith(args ...interface{}) string {
    ok := false
    var s string
    if len(args) == 1 {
        s, ok = args[0].(string)
    }
    if !ok {
        s = fmt.Sprint(args...)
    }
    // find the @ symbol
    substrs := strings.Split(s, "@")
    if len(substrs) != 2 {
        return s
    }
    // replace the @ by " at "
    return (substrs[0] + " at " + substrs[1])
}
 
func main() {
    f1 := Friend{Fname: "minux.ma"}
    f2 := Friend{Fname: "xushiwei"}
    t := template.New("fieldname example")
    t = t.Funcs(template.FuncMap{"emailDeal": EmailDealWith})
    t, _ = t.Parse(`hello {{.UserName}}!
                {{range .Emails}}
                    an emails {{.|emailDeal}}
                {{end}}
                {{with .Friends}}
                {{range .}}
                    my friend name is {{.Fname}}
                {{end}}
                {{end}}
                `)
    p := Person{UserName: "Astaxie",
        Emails:  []string{"astaxie@beego.me", "astaxie@gmail.com"},
        Friends: []*Friend{&f1, &f2}}
    t.Execute(os.Stdout, p)
}
```

# 参考

* [https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/07.4.md](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/07.4.md)
* [https://blog.gopheracademy.com/advent-2017/using-go-templates](https://blog.gopheracademy.com/advent-2017/using-go-templates)


