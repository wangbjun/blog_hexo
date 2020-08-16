---
title: 浅谈Golang里面的map应用
date: 2020-07-28 23:00:00
tags: Golang
category: Golang
---
在实际编程开发中，无论在任何语言里面，除了数组之外，最常用的数据结构莫非map，map是一种数据结构，在很多语言里面都内置了map类型，map一般被翻译成映射或者地图。从数据结构定义上，map是一组键-值对的集合，例如：
```
张三： 18
李四： 20
王五： 25
```
map强调的是一一对应，在具体实现上，不同语言略有差异，比如弱类型语言里面就不会限制key或value的类型，但是在强类型语言比如Go里面，键值的类型是需要声明限定。

但是需要注意的是，map结构的key是唯一的，但是value可以重复。

说到实现原理，大多数语言里面map都是基于hash结构实现，为了解决hash冲突问题，又引入开放地址法和链表结构，Go也不例外，这一点大家简单了解一下即可，并不影响咱们使用，下面咱就结合Go语言来看看map的常见用法。

<!--more-->

## 1.基本语法
```go
package main

import "fmt"

func main() {
    m := make(map[string]string, 10) // 使用make初始化，第二个参数可选
    m["1"] = "1"
    m["2"] = "2"
    m["3"] = "3"

    m["1"] = "1+1"

    fmt.Printf("%v\n", m)
    fmt.Printf("%d\n", len(m))
}
```
map的使用特别简单，使用make初始化，和Go里面大多数类型一样，也可以使用字面量初始化，这里不多说。

## 2.快速查找
Hash这种数据结构的查找时间复杂度是O(1)，map也继承了这个优点，所以当你知道key，想在map里面查找一个元素，那是相当快滴。
```go
func find() {
    m := make(map[string]int, 10)
    m["我"] = 10
    m["爱"] = 11
    m["Golang"] = 12

    fmt.Printf("%d\n", m["我"])
    fmt.Printf("%d\n", m["不存在"]) // 不存在的key会返回一个value类型的零值

    v, ok := m["不存在"] // 如果key存在，ok返回true
    if ok {
        fmt.Printf("%d\n", v)
    }else {
        fmt.Println("key 不存在")
    }
}
```
值得注意的是，如果你访问一个不存在的key，默认会返回零值，所以为了靠谱的判断map里面是否包含某key，可以使用 ``` v, ok := m[key]```这种写法。

## 3.去重计数
由于map的key是唯一的，如果你给map里面的某个key多次赋值，会以最后一次为准，所以利用这个特性，我们可以用来给一组数据做去重，顺便计数，效率也很高，举例说明：
```go
package main

import "fmt"

func main() {
    data := []string{"111","222","333","123","111","333","444","345","111","222","145","456"}

    m := make(map[string]int)

    for _, v := range data {
        m[v]++
    }

    fmt.Printf("%v\n", m) // map[111:3 123:1 145:1 222:2 333:2 345:1 444:1 456:1]
}
```
下面开始划重点，map的key不仅仅可以是普通类型，也可以是结构体类型，所以我们不仅可以针对普通类型做去重，可以对结构体做去重，而结构体可以有多个成员，举个例子，根据**名字+年龄**对数据进行去重，这个用法很多人可能不了解。
```go
package main

import "fmt"

type Item struct {
    name string
    age  int
}

func main() {
    var (
        names = []string{"张三", "李四", "王五", "小六", "王五"}
        ages  = []int{20, 25, 25}
    )

    m := make(map[Item]int)
    for _, i := range names {
        for _, j := range ages {
            m[Item{
                name: i,
                age:  j,
            }]++
        }
    }
    fmt.Printf("%v\n", m)
}
```
结果如下： map[{小六 20}:1 {小六 25}:2 {张三 20}:1 {张三 25}:2 {李四 20}:1 {李四 25}:2 {王五 20}:2 {王五 25}:4]

## 4.同步map
在Go的开发中，map经常也会拿来当key-value临时缓存使用，虽然是单机缓存，但是效率也很高，不过当多个协程同时读写map的时候存在一个并发问题。
```go
package main

import (
    "time"
)

func main() {
    m := make(map[int]int)
    go func() {
        for {
            for i := 1; i < 10; i++ {
                m[i] = 1
            }
        }
    }()
    go func() {
        for {
            for i := 1; i < 10; i++ {
                println(m[i])
            }
        }
    }()
    time.Sleep(time.Second * 10)
}
```
上面的代码实际运行中，有大概率会出现**fatal error: concurrent map read and map write**，除非你设置```runtime.GOMAXPROCS(1)```，这相当于开启了单线程模式。

为了解决问题，有两种方案，一种是自己手动加锁，利用sync包里面的排它锁，在读写操作之前先加锁，操作完之后解锁，示例：
```go
go func() {
    for {
        for i := 1; i < 10; i++ {
            locker.Lock() // 加锁
            m[i] = 1
            locker.Unlock() // 解锁
        }
    }
}()
```

另一种方案则是使用**sync.Map**,这个数据结构提供了 load、store这2个方法用于读写map，需要注意的是这个2个方法的参数都是 **interface{}** 类型，所以在使用的时候需要做断言处理，稍微有点不便，这也是没办法，因为Go没有泛型。
```go
m := sync.Map{}
go func() {
    for {
        for i := 1; i < 10; i++ {
            m.Store(i, 1)
        }
    }
}()
go func() {
    for {
        for i := 1; i < 10; i++ {
            load, ok := m.Load(i)
            if ok {
                println(load.(int))
            }
        }
    }
}()
time.Sleep(time.Second * 10)
```
