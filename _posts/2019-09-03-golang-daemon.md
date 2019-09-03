---
title: golang 后台执行程序
date: 2019-09-03 09:00:00
categories:
- GoLang
tags:
- 开发
description: golang中在后台执行程序的一种方式
---

```go
package main
 
import (
    "flag"
    "fmt"
    "io/ioutil"
    "os"
    "os/exec"
    "path/filepath"
    "strconv"
)
 
var godaemon = flag.Bool("d", false, "")
 
func init() {
 
    if !flag.Parsed() {
        flag.Parse()
    }
    if !*godaemon {
        //不允许修改程序名
        binname := filepath.Base(os.Args[0])
        if binname != "ITEFSDataLoader" {
            fmt.Println("this bin name must be ITEFSDataLoader")
            os.Exit(1)
        }
 
        //尝试读取文件锁中进程号
        pid, err := ioutil.ReadFile("/var/run/" + binname + ".pid")
        //如果读取进程号成功
        if err == nil {
            //判断当前进程号目录是否在/proc/目录下存在
            fileinfo, err := os.Stat("/proc/" + string(pid))
            //存在表示当前进程运行中,退出
            if err == nil && fileinfo.IsDir() {
                fmt.Printf("%s[%s] is runing\n", binname, string(pid))
                os.Exit(0)
            }
        } else if !os.IsNotExist(err) {
            //读取进程号失败
            //错误是 pid文件不存在 以外的错误，退出
            fmt.Println(err)
            os.Exit(1)
        }
        //pid文件不存在，或者已经退出，继续
 
        args := os.Args[1:]
        args = append(args, "-d", "true")
        cmd := exec.Command(os.Args[0], args...)
        cmd.Start()
        pidno := strconv.Itoa(cmd.Process.Pid)
        // 写pid文件
        err = ioutil.WriteFile("/var/run/"+binname+".pid", []byte(pidno), os.ModePerm)
        if err != nil {
            fmt.Println(err)
            os.Exit(1)
        }
        fmt.Println("[PID]", pidno)
        os.Exit(0)
    }
 
}
```

