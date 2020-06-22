---
title: 浅谈Go定时器应用
date: 2020-06-18 23:00:00
tags: 
- Golang
category: Golang
---
很多人都用过Linux的定时任务，举个例子，我们有一个任务需要每个1分钟运行一次，如果使用Linux的crontab可以用 ```* * * * *```表示。

在Go里面自带类似的功能实现，我们称之为定时器，非常简单易用，举个例子，这里使用time包里面的ticker实现：
```go
package main

import (
    "log"
    "time"
)

func main() {
    log.Println("等待10秒钟。。。")

    ticker := time.NewTicker(10 * time.Second)
    <-ticker.C

    log.Println("执行")
}
```
执行结果如下：
```
2020/06/18 21:53:31 等待10秒钟。。。
2020/06/18 21:53:41 执行
```
<!--more-->

## 1.for...select
假设我们想实现一种定时执行的效果，只要把它放到一个for循环里面即可：
```go
func main() {
   ticker := time.NewTicker(10 * time.Second)
    for {
        <-ticker.C
        log.Println("执行")
    }
}
```
但是我们通常更常见的一种写法是配合select使用，其实如果只是一个定时任务，这2者没什么差异。
```go
func main() {
    ticker := time.NewTicker(10 * time.Second)
    for {
        select {
        case <-ticker.C:
            log.Println("执行")
        }
    }
}
```
## 2.注意的坑
很多人以为这个定时任务和Linux的一样，是```定时```执行的,举个例子，我们有2个定时器，一个10s执行一次，1个20s执行一次，一般是这样写：
```go
func main() {
    t1 := time.NewTicker(10 * time.Second)
    t2 := time.NewTicker(20 * time.Second)
    for {
        select {
        case <-t1.C:
            log.Println("执行10s")
            // time.Sleep(15*time.Second)
        case <-t2.C:
            log.Println("执行20s")
        }
    }
}
```
但是，假如第一个10s的任务执行耗时超过了20s，那么多第二个定时任务就无法```按时```执行了。
```
2020/06/18 22:23:25 执行10s
2020/06/18 22:23:50 执行20s
2020/06/18 22:23:50 执行10s
2020/06/18 22:24:15 执行10s
2020/06/18 22:24:40 执行20s
2020/06/18 22:24:40 执行10s
```
究其原因，是因为select在同一时间只能选择一个分支执行，当你前面那个分支任务“阻塞“住了，就无法抽出空来执行第二个任务，即使已经过了定时器设定的时间。

熟悉Linux的crontab定时任务的人知道，它不会管你上一个任务有没有执行完成，会“非常准时”的执行。

在上面这个例子，由于select是随机选择一个分支执行，所以你会发现这2个任务并不是交互执行，比如上面的例子里面，10s执行了2次。

所以严格依赖定时的任务得考虑任务本身执行耗费的时间，不然可能会得到你预想之外的结果，其实有一个非常简单的方法可以解决这个问题，我们只要把需要执行的任务用go启动一个协程去执行即可。

## 3.timer延迟执行
细心的人注意了，time包里面还有一个timer，它和这个ticker有啥区别呢？tick原指机械钟表指针转动的声音，它表示的意思是每隔固定的时间段做一件事。

而timer则是“定时炸弹”的定时，只不过它只执行一次（定时炸弹炸完就没），某些时候是和之前的ticker一样，比如延迟执行：
```go
package main

import (
    "log"
    "time"
)

func main() {
    log.Println("等待10秒钟。。。")

    timer := time.NewTimer(10 * time.Second)
    <-timer.C

    log.Printf("执行")
}
```
有人说，这和我写个```time.Sleep(10 * time.Second)```有啥区别？这个。。。有待研究！

这个包里面还有2个等价的简化写法：
```go
func main() {
    log.Println("等待10秒钟。。。")

    <-time.After(10*time.Second)

    //time.AfterFunc(10*time.Second, func() {
    //   log.Printf("执行")
    //})

    log.Printf("执行")
}
```

那这个timer存在的意义到底在哪呢？

首先，这个timer是一次性的，所以没法向之前那样放到for循环里面来实现定时执行的功能。

其次，除了可以作为一个一次性延迟执行功能之外，还可以作为一个实现超时的功能，举个例子，从一个管道中等待获取数据。
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan int)
    t := time.NewTimer(5 * time.Second)
    select {
    case <-ch1:
        t.Stop()
        //do something....
    case <-t.C:
        fmt.Println("5s已经过去，任务超时")
    }
}
```

关于Go定时器的介绍就说这么多，下次再说说其内部实现原理。