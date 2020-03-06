---
title: 浅谈Golang协程
date: 2018-12-10 12:05:45
tags: Golang
category: 编程开发
---

## 前言
学习和使用golang也有一段时间了，golang最近2年在国内很火，提起golang和其它语言最大区别莫过于协程，不过咱今天先不说协程，我先说一下自己的一些理解。

对c熟悉的人应该对go不陌生，它们都属于强类型静态编译型语言，在语法上和PHP这种弱类型动态解释型语言不一样，虽然差异很大，但是基本语法都是差不多，掌握一种语言之后再去学其它语言语法不是什么大问题。

在IT行业，编程语言之争一直是个很热闹的话题，编程语言之间的区别不仅仅在于语法和特性，语法只是表达编程思想的方式，一个编程语言的背后往往是其强大的生态圈，比如c语言之所以经久不衰，那是因为它几乎可以认为是创世纪语言，是当代编程的起点，而PHP则以快速处理文本，快速搭建web网站出名，JS则是浏览器编程的唯一选择，Python拥有的科学计算库是其它语言没有的。

说到go的优点，一般都集中在静态编译、毫秒级GC、简洁、并发并行等特性上面，go是2008年诞生的，由C语言之父设计，相对其它语言来说比较年轻，可以说在设计之初吸收了各大语言的优点。

<!--more-->

## 协程到底是什么东西？
说到go必须得说协程，先说说为什么需要协程，都说go是为并发编程而生，指的就是go很容易写出高并发的程序，现代计算机硬件早已步入多核时代，前段时间AMD刚刚发布最新的锐龙3代，作为民用级的CPU现在已达到16核32线程，然而大部分编程语言依然弱智，只能利用单核性能，传说中一核有难多核围观...

但是操作系统提供了多进程的能力，除了多进程之外，还有一个叫多线程，线程和进程区别不大，线程是程序执行的最小单位,一个进程可以有多个线程，编程语言可以使用多进程或多线程利用多核CPU的能力，然而现实并不是那么简单...

进程和线程都可以解决多核CPU利用率的问题，比如PHP就整出来一个fpm，采用了master-worker模型，实际上采用多进程解决并发问题，已经非常不错了，但是依然存在问题，支持不了太高的并发。

现在的Linux和Windows都是分时复用的多任务操作系统，上面跑着很多程序，所以操作系统需要在不同进程之间切换，这时候就产生了CPU上下文切换，具体技术细节咱可以不了解，但是存在的问题就是切换的时候非常消耗资源，默认情况下Linux只可以创建1024个进程，虽然可以修改，但是一旦进程或线程数过多，CPU的时间基本上都浪费在上下文切换上面了，何谈高效？

可见，多进程和多线程并不是很完美，对于编程来说，难度非常大，所以目前只有Java有比较好的多线程模型，PHP虽然有相关扩展，但是很少有人使用，JS压根不支持！

但是并不是必须使用多进程或多线程才可以实现高并发，很多时候，特别是web相关应用，当你读取文件或者调用API都会产生IO，但是由于计算机硬盘、网络传输速度比较慢，CPU就会一直在那等...时间就浪费了！后来有人想，既然在等IO，你就把CPU让出来让其它人用啊，当硬盘数据读取到、接口返回数据的时候我通知你一声就行了，这就是异步非阻塞IO，JS目前使用就是这种模型，Golang的协程也会用到。

在我理解，go的协程是为了解决多核CPU利用率问题，go语言层面并不支持多进程或多线程，但是协程更好用，协程被称为用户态线程，不存在CPU上下文切换问题，效率非常高。

## 实例

### 1.Hello World
```go
package main

func main() {
    go say("Hello World")
}

func say(s string) {
    println(s)
}
```
go启动协程的方式就是使用关键字go，后面一般接一个函数或者匿名函数，但是如果你运行上面第一段代码，你会发现什么结果都没有，what？？？

这至少说明你代码写的没问题，当你使用go启动协程之后,后面没有代码了，这时候主线程结束了，这个协程还没来得及执行就结束了... 聪明的小伙伴会想到，那我主线程先睡眠1s等一等？ Yes, 在main代码块最后一行加入：
```go
time.Sleep(time.Second*1) # 表示睡眠1s
```
再次运行，打印出Hello World，1s后程序终止！

### 2.WaitGroup
上面睡眠这种做法肯定是不靠谱的，WaitGroup可以解决这个问题, 代码如下:
```go
package main

import (
    "sync"
)
var wg = sync.WaitGroup{}

func main() {
    wg.Add(1)
    go say("Hello World")
    wg.Wait()
}

func say(s string) {
    println(s)
    wg.Done()
}
```
简单说明一下用法，var 是声明了一个全局变量 wg，类型是sync.WaitGroup，wg.add(1) 是说我有1个协程需要执行，wg.Done 相当于 wg.Add(-1) 意思就是我这个协程执行完了。wg.Wait() 就是告诉主线程要等一下，等协程都执行完再退出。

### 3.并发还并行?
当你同时启动多个协程的时候，会怎么执行呢？
```go
package main

import (
    "strconv"
    "sync"
)

var wg = sync.WaitGroup{}

func main() {
    wg.Add(5)
    for i := 0; i < 5; i++ {
        go say("Hello World: " + strconv.Itoa(i))
    }
    wg.Wait()
}

func say(s string) {
    println(s)
    wg.Done()
}
```
如果去掉go，直接在循环里面调用这个函数5次，毫无疑问会一次输出 Hello World: 0 ~ 4, 但是在协程里面，输出的顺序是无序的，看上去像是“同时执行”，其实这只是并发。

有一个问题，上面的例子里面是并发还并行呢？

首先，我们得区分什么是并发什么是并行，举个比较熟悉的例子，并发就是一个锅同时炒2个菜，2个菜来回切换，并行就是你有多个锅，每个锅炒不同的菜，多个锅同时炒！

对于计算机来说，这个锅就是CPU，单核CPU同一时间只能执行一个程序，但是CPU却可以在不同程序之间快速切换，所以你在浏览网页的同时还可以听歌！但是多核CPU就不一样了，操作系统可以一个CPU核心用来浏览网页，另一个CPU核心拿来听歌，所以多核CPU还是有用的。

但是对于单一程序来说，基本上是很难利用多核CPU的，主要是编程实现非常麻烦，这也是为什么很多人都说多核CPU是一核有难多核围观...特别是一些比较老的程序，人家在设计的时候压根没考虑到多核CPU，毕竟10年前CPU还没有这么多核心。

回到上面的例子，如果当前CPU是单核，那么上面程序就是并发执行，如果当前CPU是多核，那就是并行执行，结果都是一样的，如何证明请看下面的例子：
```go
package main

import (
    "runtime"
    "strconv"
)

func main() {
    runtime.GOMAXPROCS(1)
    for i := 0; i < 5; i++ {
        go say("Hello World: " + strconv.Itoa(i))
    }
    for {
    }
}

func say(s string) {
    println(s)
}
```
默认情况下，最新的go版本协程可以利用多核CPU，但是通过runtime.GOMAXPROCS() 我们可以设置所需的核心数（其实并不是CPU核心数），在上面的例子我们设置为1，也就是模拟单核CPU，运行这段程序你会发现无任何输出，如果你改成2，你会发现可以正常输出。

这段程序逻辑很简单，使用一个for循环启动5个协程，然后写了一个for死循环，如果是单核CPU，当运行到for死循环的时候，由于没有任何io操作（或者能让出CPU的操作），会一直卡在那里，但是如果是多核CPU，go协程就会调用其它CPU去执行。

如果你在for循环里面加入一个sleep操作，比如下面这样：
```go
for {
    time.Sleep(time.Second)
}
```
运行上面代码，你会发现居然可以正常输出，多次运行你会发现其结果一直是从0到4，变成顺序执行了！这也说明单核CPU只能并发，不能并行！

### 4.channel
channel，又叫管道，在go里面的管道是协程之间通信的渠道，类似于咱们常用的消息队列。在上面的例子里面我们是直接打印出来结果，假如现在的需求是把输出结果返回到主线程呢？
```go
package main

import (
    "strconv"
)

func main() {
    var result = make(chan string)
    for i := 0; i < 5; i++ {
        go say("Hello World: "+strconv.Itoa(i), result)
    }
    for s := range result {
        println(s)
    }
}

func say(s string, c chan string) {
    c <- s
}
```
简单说明一下，这里就是实例化了一个string类型的管道，在调用函数的时候会把管道当作参数传递过去，然后在调用函数里面我们不输出结果，然后把结果通过管道返还回去，然后再主线程里面我们通过for range循环依次取出结果！

结果如下，但是这个程序是有bug的，在程序的运行的最后会出现一个fatal error，提示所有的协程都进入睡眠状态，死锁！
```
Hello World: 4
Hello World: 1
Hello World: 0
Hello World: 2
Hello World: 3
fatal error: all goroutines are asleep - deadlock!
```
go的管道默认是阻塞的(假如你不设置缓存的话)，你那边放一个，我这头才能取一个，如果你那边放了东西这边没人取，程序就会一直等下去，死锁了，同时，如果那边没人放东西，你这边取也取不到，也会发生死锁！

如何解决这个问题呢？标准的做法是主动关闭管道，或者你知道你应该什么时候关闭管道, 当然你结束程序管道自然也会关掉！针对上面的演示代码，可以这样写：
```go
var i = 0
for s := range result {
    println(s)
    if i >= 4 {
        close(result)
    }
    i++
}
```
因为我们明确知道总共会输出5个结果，所以这里简单做了一个判断，大于5就关闭管道退出for循环，就不会报错了！虽然丑了点，但是能用

### 5.生产者消费者模型
利用channel和协程，我们可以非常容易的实现了一个生产者消费者模型，代码如下：
```go
package main

import (
    "strconv"
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan int)
    ch2 := make(chan string)
    go pump1(ch1)
    go pump2(ch2)
    go suck(ch1, ch2)
    time.Sleep(time.Duration(time.Second*30))
}

func pump1(ch chan int) {
    for i := 0; ; i++ {
        ch <- i * 2
        time.Sleep(time.Duration(time.Second))
    }
}

func pump2(ch chan string) {
    for i := 0; ; i++ {
        ch <- strconv.Itoa(i+5)
        time.Sleep(time.Duration(time.Second))
    }
}

func suck(ch1 chan int, ch2 chan string) {
    chRate := time.Tick(time.Duration(time.Second*5)) // 定时器
    for {
        select {
        case v := <-ch1:
            fmt.Printf("Received on channel 1: %d\n", v)
        case v := <-ch2:
            fmt.Printf("Received on channel 2: %s\n", v)
        case <-chRate:
            fmt.Printf("Log log...\n")
        }
    }
}
```
输出结果如下：
```
Received on channel 1: 0
Received on channel 2: 5
Received on channel 2: 6
Received on channel 1: 2
Received on channel 1: 4
Received on channel 2: 7
Received on channel 1: 6
Received on channel 2: 8
Received on channel 2: 9
Received on channel 1: 8
Log log...
Received on channel 2: 10
Received on channel 1: 10
Received on channel 1: 12
Received on channel 2: 11
Received on channel 2: 12
Received on channel 1: 14
```
这个程序建立了2个管道一个传输int，一个传输string，同时启动了3个协程，前2个协程非常简单，就是每隔1s向管道输出数据，第三个协程是不停的从管道取数据，和之前的例子不一样的地方是，pump1 和 pump2是2个不同的管道，通过select可以实现在不同管道之间切换，哪个管道有数据就从哪个管道里面取数据，如果都没数据就等着，还有一个定时器功能可以每隔一段时间向管道输出内容！而且我们可以很容易启动多个消费者。

## 应用
### 1.Web应用
使用go自带的http库几行代码就可以启动一个http server，代码如下：
```go
http.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {
        _, _ = fmt.Fprintln(writer, "Hello World")
    })

_ = http.ListenAndServe("127.0.0.1:8080", nil)
```
虽然简单，但是非常高效，因为其底层使用了go协程，对于每一个请求都会启动一个协程去处理，所以并发可以轻轻松松达到上万QPS。

### 2.并发编程
举一个非常简单的例子，假设我们在业务里面需要从3个不同的数据库获取数据，每次耗时100ms，正常写法就是从上到下依次执行，总耗时300ms，但是使用协程这3个操作可以“同时”进行，耗时大大减少。

几乎所有IO密集型的应用，都可以利用协程提高速度，提高程序并发能力，不必把CPU时间浪费在等待的过程中，同时还可以充分利用多核CPU的计算能力。
