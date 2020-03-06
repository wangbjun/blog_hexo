---
title: 浅谈Golang的recover异常处理机制
date: 2019-11-10 20:22:04
tags: Golang
category: 编程开发
---

## 1.error
Golang被诟病非常多的一点就是缺少强大方便的异常处理机制，大部分高级编程语言，比如Java、PHP、Python等都拥有一种try catch机制，这种异常捕获机制可以非常方便的处理程序运行中可能出现的各种意外情况。

严格来说，在Go里面，错误和异常是2种不同的类型，错误一般是指程序产生的逻辑错误，或者意料之中的意外情况，而且异常一般就是panic，比如角标越界、段错误。

<!--more-->

对于错误，Golang采用了一种非常原始的手段，我们必须手动处理可能产生的每一个错误，一般会把错误返回给调用方，下面这种写法在Go里面十分常见：
```go
package main
import (
    "errors"
    "fmt"
)

func main() {
    s, err := say()
    if err != nil {
        fmt.Printf("%s\n", err.Error())
    } else {
        fmt.Printf("%s\n", s)
    }
}

func say() (string, error) {
    // do something
    return "", errors.New("something error")
}
```
这种写法最大的问题就是每一个error都需要判断处理，非常繁琐，如果使用try catch机制，我们就可以统一针对多个函数调用可能产生的错误做处理，节省一点代码和时间。不过咱们今天不是来讨论Go的异常错误处理机制的，这里只是简单说一下。

## 2.panic
一般错误都是显示的，程序明确返回的，而异常往往是隐示的，不可预测的，比如下面的代码：
```go
package main

import "fmt"

func main() {
    fmt.Printf("%d\n", cal(1,2))
    fmt.Printf("%d\n", cal(5,2))
    fmt.Printf("%d\n", cal(5,0)) //panic: runtime error: integer divide by zero 
    fmt.Printf("%d\n", cal(9,5))
}

func cal(a, b int) int {
    return a / b
}
```
在执行第三个计算的时候会发生一个panic，这种错误会导致程序退出，下面的代码的就无法执行了。当然你可以说这种错误理论上是可以预测的，我们只要在cal函数内部做好处理就行了。

然而实际开发中，会发生panic的地方可能特别多，而且不是这种一眼就能看出来的，在Web服务中，这样的panic会导致整个Web服务挂掉，特别危险。

## 3.recover
虽然没有try catch机制，Go其实有一种类似的recover机制，功能弱了点，用法很简单：
```go
package main

import "fmt"

func main() {
    fmt.Printf("%d\n", cal(1, 2))
    fmt.Printf("%d\n", cal(5, 2))
    fmt.Printf("%d\n", cal(5, 0))
    fmt.Printf("%d\n", cal(9, 2))
}

func cal(a, b int) int {
    defer func() {
        if err := recover(); err != nil {
            fmt.Printf("%s\n", err)
        }
    }()
    return a / b
}
```
首先，大家得理解defer的作用，简单说defer就类似于面向对象里面的析构函数，在这个函数终止的时候会执行，即使是panic导致的终止。

所以，在cal函数里面每次终止的时候都会检查有没有异常产生，如果产生了我们可以处理，比如说记录日志，这样程序还可以继续执行下去。

## 4.注意的坑
一般defer recover这种机制经常用在常驻进程的应用，比如Web服务，在Go里面，每一个Web请求都会分配一个goroutine去处理，在没有做任何处理的情况下，假如某一个请求发生了panic，就会导致整个服务挂掉，这是不可接受的，所以在Web应用里面必须使用recover保证即使某一个请求发生错误也不影响其它请求。

这里我使用一小段代码模拟一下：
```go
package main

import (
    "fmt"
)

func main() {
    requests := []int{12, 2, 3, 41, 5, 6, 1, 12, 3, 4, 2, 31}
    for n := range requests {
        go run(n) //开启多个协程
    }

    for {
        select {}
    }
}

func run(num int) {
    //模拟请求错误
    if num%5 == 0 {
        panic("请求出错")
    }
    fmt.Printf("%d\n", num)
}
```
上面这段代码无法完整执行下去，因为其中某一个协程必然会发生panic，从而导致整个应用挂掉，其它协程也停止执行。

解决方法和上面一样，我们只需要在run函数里面加入defer recover，整个程序就会非常健壮，即使发生panic，也会完整的执行下去。
```go
func run(num int) {
    defer func() {
        if err := recover();err != nil {
            fmt.Printf("%s\n", err)
        }
    }()
    if num%5 == 0 {
        panic("请求出错")
    }
    fmt.Printf("%d\n", num)
}
```

上面的代码只是演示，真正的坑是：如果你在run函数里面又启动了其它协程，这个协程发生的panic是无法被recover的，还是会导致整个进程挂掉,我们改造了一下上面的例子：
```go
func run(num int) {
    defer func() {
        if err := recover(); err != nil {
            fmt.Printf("%s\n", err)
        }
    }()
    if num%5 == 0 {
        panic("请求出错")
    }
    go myPrint(num)
}

func myPrint(num int) {
    if num%4 == 0 {
        panic("请求又出错了")
    }
    fmt.Printf("%d\n", num)
}
```
我在run函数里面又通过协程的方式调用了另一个函数，而这个函数也会发生panic，你会发现整个程序也挂了，即使run函数有recover也没有任何作用，这意味着我们还需要在myPrint函数里面加入recover。但是如果你不使用协程的方式调用myPrint函数，直接调用的话还是可以捕获recover的。

总结一下就是defer recover这种机制只是针对当前函数和以及直接调用的函数可能产生的panic，它无法处理其调用产生的其它协程的panic，这一点和try catch机制不一样。

理论上讲，所有使用协程的地方都必须做defer recover处理，这样才能保证你的应用万无一失，不过开发中可以根据实际情况而定，对于一些不可能出错的函数加了还影响性能。

Go的Web服务也是一样，默认的recover机制只能捕获一层，如果你在这个请求的处理中又使用了其它协程，那么必须非常慎重，毕竟只要发生一个panic，整个Web服务就会挂掉。

最后，总结一下，Go的异常处理机制虽然没有很多其它语言高效，但是基本上还是能满足需求，目前官方已经在着完善这一点，Go2可能会见到。
