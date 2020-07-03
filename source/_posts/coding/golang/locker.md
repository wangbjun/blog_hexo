---
title: 浅谈Golang锁的应用（sync包）
date: 2020-07-02 22:00:00
tags: 
- Golang
- 锁
category: Golang
---
今天谈一下锁，以及Go里面Sync包里面自带的各种锁，说到锁这个概念，在日常生活中，锁是为了保护一些东西，比如门锁、密码箱锁，可以理解对资源的保护。在编程里面，锁也是为了保护资源，比如说对文件加锁，同一时间只也许一个用户修改，这种锁一般叫作文件锁。

实际开发中，锁又可分为互斥锁（排它锁）、读写锁、共享锁、自旋锁，甚至还有悲观锁、乐观锁这种说法。在Mysql数据库里面锁的应用更多，比如行锁、表锁、间隙锁，有点眼花缭乱。抛开这些概念，在编程领域，锁的本质是为了解决并发情况下对数据资源的访问问题，如果我们不加锁，并发读写一块数据必然会产生问题，如果直接加个互斥锁问题是解决了，但是会严重影响读写性能，所以后面又产生了更复杂的锁机制，在数据安全性和性能之间找到最佳平衡点。

<!--more-->

正常来说，只有在并发编程下才会需要锁，比如说多个线程（在Go里面则是协程）同时读写一个文件，下面我以一个文件为例，来解释这几种锁的概念：

如果我们使用互斥锁，那么同一时间只能由一线程去操作（读或写），这就是像是咱们去上厕所，一个坑位同一时间只能蹲一个人，这就是厕所门锁的作用。

如果我们使用读写锁，意味着可以同时有多个线程读取这个文件，但是写的时候不能读，并且只能由一个线程去写。这个锁实际上是互斥锁的改进版，很多时候我们之所以给文件加锁是为了避免你在写的过程中有人读到了脏数据。

如果我们使用共享锁，根据我查到资料，这种叫法大多数是源自MySQL事务里面的锁概念，它意味着只能读数据，并不能修改数据。

如果我们使用自旋锁，则意味着当一个线程在获取锁的时候，如果锁已经被其它线程获取，那么该线程将循环等待，然后不断的判断锁是否能够被成功获取，直到获取到锁才会退出循环。

这些锁的机制在Go里面有什么应用呢，下面大家一起看看Go标准库里面sync包提供的一些非常强大的基于锁的实现。

## 1.文件锁
文件锁和sync包没关系，这里面只是顺便说一下，举个例子，磁盘上面有一个文件，必须保证同一时间只能由一个人打开，这里的同一时间是指操作系统层面的，并不是指应用层面，文件锁依赖于操作系统实现。

在C或PHP里面，文件锁会使用一个flock的函数去实现，其实Go里面也类似：
```go
func main() {
    var f = "/var/logs/app.log"
    file, err := os.OpenFile(f, os.O_RDWR, os.ModeExclusive)
    if err != nil {
        panic(err)
    }
    defer file.Close()

    // 调用系统调用加锁
    err = syscall.Flock(int(file.Fd()), syscall.LOCK_EX|syscall.LOCK_NB)
    if err != nil {
        panic(err)
    }
    defer syscall.Flock(int(file.Fd()), syscall.LOCK_UN)
    // 读取文件内容
    all, err := ioutil.ReadAll(file)
    if err != nil {
        panic(err)
    }

    fmt.Printf("%s", all)
    time.Sleep(time.Second * 10) //模拟耗时操作
}
```
需要说明一下，Flock函数第一个参数是文件描述符，第二个参数是锁的类型，分为LOCK_EX（排它锁）、LOCK_SH（读共享锁）、LOCK_NB（遭遇锁的表现，遇到排它锁的时候默认会被阻塞，NB即非阻塞，直接返回Error）、LOCK_UN（解锁）。

如果这时候你打开另外一个终端再次运行这个程序你会发现报错信息如下：
```
panic: resource temporarily unavailable
```
文件锁保证了一个文件在操作系统层面的数据读写安全，不过实际应用中并不常见，毕竟大部分时候我们都是使用数据库去做数据存储，极少使用文件。

## 2.sync.Mutex
下面我所说的这些锁都是应用级别的锁，位于Go标准库sync包里面，各有个的应用场景。

这是一个标准的互斥锁，平时用的也比较多，用法也非常简单，lock用于加锁，unlock用于解锁，配合defer使用，完美。

为了更好的展示锁的应用，这个举一个没有实际意义的例子,给一个int变量做加法，用2个协程并发的去做加法。
```go
var i int

func main() {
    go add(&i)
    go add(&i)

    time.Sleep(time.Second * 3)

    println(i)
}

func add(i *int) {
    for j := 0; j < 10000; j++ {
        *i = *i + 1
    }
}
```
我们想要得到得正常结果是20000，然而实际上并不是，其结果是不固定的，很可能少于20000，大家多运行几次便可得知。
>假设你多加一行 runtime.GOMAXPROCS(1)，你会发现结果一直是正确的，这是为什么呢？

用一个比较理论的说法，这是因为产生了数据竞争（data race）问题，在Go里面我们可以在go run后面加上 **-race** 来检测数据竞争，结果会告诉你在哪一行产生的，非常实用。
```
go run -race main.go
==================
WARNING: DATA RACE
Read at 0x00000056ccb8 by goroutine 7:
  main.add()
      main.go:23 +0x43
Previous write at 0x00000056ccb8 by goroutine 6:
  main.add()
       main.go:23 +0x59
Goroutine 7 (running) created at:
  main.main()
       main.go:14 +0x76
Goroutine 6 (running) created at:
  main.main()
       main.go:13 +0x52
==================
20000
Found 1 data race(s)
exit status 66
```

解决这个问题，有多种解法，我们当然可以换个写法，比如说用chan管道去做加法（chan底层也用了锁），实际上在Go里面更推荐去使用chan解决数据同步问题，而不是直接用锁机制。

在上面的这个例子里面我们需要在add方法里面写,每次操作之前lock，然后unlock：
```go
func add(i *int) {
    for j := 0; j < 10000; j++ {
        s.Lock()
        *i = *i + 1
        s.Unlock()
    }
}
```

## 3.sync.RWMutex
读写锁是互斥锁的升级版，它最大的优点就是支持多读，但是读和写、以及写与写之间还是互斥的，所以比较适合读多写少的场景。

它的实现里面有5个方式：
```go
func (rw *RWMutex) Lock()
func (rw *RWMutex) RLock()
func (rw *RWMutex) RLocker() Locker
func (rw *RWMutex) RUnlock()
func (rw *RWMutex) Unlock()
```
其中Lock()和Unlock()用于申请和释放写锁,RLock()和RUnlock()用于申请和释放读锁，RLocker()用于返回一个实现了Lock()和Unlock()方法的Locker接口。

实话说，平时这个用的真不多，主要是使用起来比较复杂，虽然在读性能上面比 **Mutex** 要好一点。

## 4.sync.Map
这个类型印象中是后来加的，最早很多人使用互斥锁来并发的操作map，现在也还有人这么写:
```go
type User struct {
    m map[string]string
    l sync.Mutex
}
```
也就是一个map配一把锁的写法，可能是这种写法比较多，于是乎官方就在标准库里面实现了一个 **sync.Map**,是一个自带锁的map，使用起来方便很多，省心。
```go
var m sync.Map

func main() {
    m.Store("1", 1)
    m.Store("2", 1)
    m.Store("3", 1)
    m.Store(4, "5") // 注意类型

    load, ok := m.Load("1")
    if ok {
        fmt.Printf("%v\n", load)
    }

    load, ok = m.Load(4)
    if ok {
        fmt.Printf("%v\n", load)
    }
}
```
需要注意的一点是这个map的key和value都是interface{}类型，所以可以随意放入任何类型的数据，在使用的时候就需要做好断言处理。

## 5.sync.Once
```go
package main

import "sync"

var once sync.Once

func main() {
    doOnce()
    doOnce()
    doOnce()
}

func doOnce() {
    once.Do(func() {
        println("one")
    })
}
```
执行结果只打印了一个 one，所以sync.Once的功能就是保证只执行一次，也算是一种锁，通常可以用于只能执行一次的初始化操作，比如说单例模式里面的懒汉模式可以用到。

## 6.sync.Cond
这个一般称之为条件锁，就是当满足某些条件下才起作用的锁，啥个意思呢？举个例子，当我们执行某个操作需要先获取锁，但是这个锁必须是由某个条件触发的，其中包含三种方式：
* 等待通知: wait,阻塞当前线程，直到收到该条件变量发来的通知

* 单发通知: signal,让该条件变量向至少一个正在等待它的通知的线程发送通知，表示共享数据的状态已经改变

* 广播通知: broadcast,让条件变量给正在等待它的通知的所有线程都发送通知

下面看一个简单的例子：
```go
package main
import (
    "sync"
    "time"
)

var cond = sync.NewCond(&sync.Mutex{})

func main() {
    for i := 0; i < 10; i++ {
        go func(i int) {
            cond.L.Lock()
            cond.Wait() // 等待通知,阻塞当前goroutine
            println(i)
            cond.L.Unlock()
        }(i)
    }

    // 确保所有协程启动完毕
    time.Sleep(time.Second * 1)

    cond.Signal()
    cond.Signal()
    cond.Signal()

    // 确保结果有时间输出
    time.Sleep(time.Second * 1)
}
```
开始我们使用for循环启动10个协程，每个协程都在等待锁，然后使用signal发送一个通知。

如果你多次运行，你会发现打印的结果也是随机从0 到 9，说明各个协程之间是竞争的，锁是起到作用的。如果把 singal 替换成 broadcast，则会打印所有结果。

讲实话，我暂时也没有发现有哪些应用场景，感觉这个应该适合需要非常精细的协程控制场景，大家先了解一下吧。

## 7.sync.WaitGroup
这个大多数人都用过，一般用来控制协程执行顺序，大家都知道如果我们直接用go启动一个协程，比如下面这个写法：
```go
go func() {
    println("1")
}()

time.Sleep(time.Second * 1) // 睡眠1s
```
如果没有后面的sleep操作，协程就得不到执行，因为整个函数结束了，主进程都结束了协程哪有时间执行，所以有时候为了方便可以直接简单粗暴的睡眠几秒，但是实际应用中不可行。这时候就可以使用 waitGroup 解决这个问题，举个例子：
```go
package main

import "sync"

var wg sync.WaitGroup

func main() {
    for i := 0; i < 10; i++ {
        wg.Add(1) // 计数+1
        go func() {
            println("1")
            wg.Done() // 计数-1，相当于wg.add(-1)
        }()
    }
    wg.Wait() // 阻塞带等待所有协程执行完毕
}
```

## 8.sync.Pool
这是一个池子，但是却是一个不怎么可靠的池子，sync.Pool 初衷是用来保存和复用临时对象，以减少内存分配，降低CG压力。

说它不可靠是指放进 Pool 中的对象，会在说不准什么时候被GC回收掉，所以如果事先 Put 进去 100 个对象，下次 Get 的时候发现 Pool 是空也是有可能的。

```go
package main

import (
    "fmt"
    "sync"
)

type User struct {
    name string
}

var pool = sync.Pool{
    New: func() interface{} {
        return User{
            name: "default name",
        }
    },
}

func main() {
    pool.Put(User{name: "name1"})
    pool.Put(User{name: "name2"})

    fmt.Printf("%v\n", pool.Get()) // {name1}
    fmt.Printf("%v\n", pool.Get()) // {name2}
    fmt.Printf("%v\n", pool.Get()) // {default name} 池子已空，会返回New的结果
}
```
从输出结果可以看到，Pool就像是一个池子，我们放进去什么东西，但不一定可以取出来（如果中间有GC的话就会被清空），如果池子空了，就会使用之前定义的New方法返回的结果。

为什么这个池子会放到sync包里面呢？那是因为它有一个重要的特性就是协程安全的，所以其底层自然也用到锁机制。

至于其应用场景，知名的Web框架Gin里面就有用到，在处理用户的每条请求时都会为当前请求创建一个上下文环境 Context，用于存储请求信息及相应信息等。Context 满足长生命周期的特点，且用户请求也是属于并发环境，所以对于线程安全的 Pool 非常适合用来维护 Context 的临时对象池。
                          

