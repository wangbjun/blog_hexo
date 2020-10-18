---
title: 详解too many open files错误
date: 2020-10-18 12:58:00
tags: linux
category: Coding
---

相信很多人在开发中过程中都遇到过这个问题： ```xxx， too many open files```，意思是打开的文件太多，默认情况下，Linux的文件最大描述符个数是1024，这一点可以在终端里面通过 ```ulimit -n```命令查询。在Linux的设计哲学里面，一切既文件，当你在操作文件、访问网络的时候都会占用文件描述符，文件描述符用尽之后什么操作都做不了，这也就是为什么咱们在编程的时候打开文件之后一定要关闭文件、网络连接用完之后也要及时关闭，可不仅仅是防止内存泄露。

<!--more-->

但是对于nginx这样的应用来说，长期占用一定描述符是必需的，如果只是少量请求或许没问题，但是在高并发情况下，比如QPS达到上万级别，默认的文件描述符个数就不够用了，不得不增大配置。

解决这个问题最简单的方式就是修改Linux系统配置，我们可以直接通过```ulimit -n 10240```这种方式修改连接符个数为10240（最大值65536），但是这只在当前终端有效，如果需要永久有效，有几种方式：

比如，在 /etc/security/limits.conf 后面加入：
```
* soft nofile 10240
* hard nofile 10240
```
也可以在 /etc/profile 里面通过 ulimit 命令修改，只不过这个是全系统有效，而不是只在当前终端有效。

---
不过当遇到这个问题的时候，我们也不要无脑的修改配置，得分情况来看，如果你是在使用nginx或者那些需要同时打开多个文件的应用遇到这个报错，直接通过ulimit修改配置就行。

但如果这是你自己编程过程中遇到的问题，那可能是你写的代码有bug: 比如说打开的文件没关闭、网络连接未关闭。其实这种情况很常见，未关闭的资源不仅会导致程序内存泄露，也会导致文件描述符被消耗殆尽，如果的你应用是常驻内存的，6万多个消耗完也只是时间问题。

下面咱们看一个简单的例子：
```go
package main

import (
    "fmt"
    "os"
    "time"
)

func main() {
    var i = 0
    for {
        open, err := os.Open("/home/jwang/Downloads/app.txt")
        if err != nil {
            panic(fmt.Sprintf("err: %s, open %d", err, i))
        }
        open.Write([]byte("abc"))
        i++
        fmt.Printf("open %d\n", i)
        time.Sleep(time.Millisecond * 10)
    }
}
```
我们在一个循环里面打开一个文件，写入内容之后，并没有主动关闭文件，在运行之前我们通过```ulimit -n 1000```设置文件描述符为1000，然后运行该程序，结果如下：
```
jwang@jun:~/demo$ go run too-many-open-files.go 
panic: err: open /home/jwang/Downloads/app.txt: too many open files, open 994

goroutine 1 [running]:
main.main()
        /home/jwang/demo/too-many-open-files.go:14 +0x172
exit status 2
```
可见，在第994次操作的时候，程序报错了，但是有时候对于一个大型应用，操作文件或者网络连接的地方非常多，怎么快速排查出是哪个地方出问题呢？

在Linux里面/proc包含了运行中程序的一些信息，其中/proc/pid/fd就可以查看进程打开的所有文件描述符，我们可以先使用ps查看正在运行中的程序的pid，然后通过ls命令查看即可：
```
jwang@jun:~$ ll /proc/17814/fd
total 0
dr-x------ 2 jwang jwang  0 10月 18 13:34 ./
dr-xr-xr-x 9 jwang jwang  0 10月 18 13:33 ../
lrwx------ 1 jwang jwang 64 10月 18 13:34 0 -> /dev/pts/18
lrwx------ 1 jwang jwang 64 10月 18 13:34 1 -> /dev/pts/18
lr-x------ 1 jwang jwang 64 10月 18 13:34 10 -> /home/jwang/Downloads/app.txt
lr-x------ 1 jwang jwang 64 10月 18 13:34 11 -> /home/jwang/Downloads/app.txt
lr-x------ 1 jwang jwang 64 10月 18 13:34 12 -> /home/jwang/Downloads/app.txt
lr-x------ 1 jwang jwang 64 10月 18 13:34 13 -> /home/jwang/Downloads/app.txt
lr-x------ 1 jwang jwang 64 10月 18 13:34 14 -> /home/jwang/Downloads/app.txt
lr-x------ 1 jwang jwang 64 10月 18 13:34 15 -> /home/jwang/Downloads/app.txt
lr-x------ 1 jwang jwang 64 10月 18 13:34 16 -> /home/jwang/Downloads/app.txt
lr-x------ 1 jwang jwang 64 10月 18 13:34 17 -> /home/jwang/Downloads/app.txt
lr-x------ 1 jwang jwang 64 10月 18 13:34 18 -> /home/jwang/Downloads/app.txt
lr-x------ 1 jwang jwang 64 10月 18 13:34 19 -> /home/jwang/Downloads/app.txt
...
```
这样我们就可以很清晰看到打开的文件，可以帮助我们快速排查到底是哪个地方有问题，非常实用。所以当下次遇到这个问题的时候不妨先排查一遍再做操作。