---
title: Golang限速算法和应用
date: 2020-06-15 22:00:00
tags: 
- Golang
category: Golang
---
今天来说说限速算法，其实这个倒不是什么神秘的事情，本文主要做一个总结，并且从Golang这么语言的角度做一个实践。

假设作为一个完全不懂算法的人，让你去实现一个限速功能（1秒内最多100次），你可能会想到的最简单方式就是记录一个开始时间，然后开始计数，当计数达到100之后限制调用，等待时间间隔达到1秒的时候重置计数器，然后重新计数，如此往复。

这种算法也是日常生活中应用广泛，比如说坐地铁，在北京，很多地铁站在早晚高峰时期都会限流，方法非常简单，地铁站门口有安检员，每过一段时间放几个人进去，虽然安检员也不会掐着表计时，但是基本上是这个规律。

这种方式被称为```计数器算法```，有些文章也称之为```固定时间窗口计数法```，因为与之对应的还有一个方法叫作```滑动时间窗口计数法```。
<!--more-->

## 1.计数器法
这种算法实现起来并不难，主要要点在于重置计数器和开始时间，我这里用Go，以一个方法的调用为例：
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    var count = 1
    var sTime = time.Now()

    for {
        if time.Now().Sub(sTime).Seconds() > 1  {
            count = 1
            sTime = time.Now()
        }
        if count > 100  {
            time.Sleep(time.Millisecond * 10)
            continue
        }
        doSomething(count)
        count++
    }
}

func doSomething(n int) {
    fmt.Printf("do something #%d\n", n)
}
```
这种算法最大问题在于临界点问题，比如说在上面这个例子里面，假设我们在前面1秒钟中最后999ms的时候打印了100个，又在第二个1秒的前1ms内又打印了100个，那么总得算起来，其在2ms内就打印了200个，超出了我们限制。

另外，还有一个问题就是1ms就会打印完100个，后面999ms都会在等待中，假设在实际应用中，比如说接口限流，假如说某个客户端请求速度比较快，那么就会瞬间消耗所有配额，导致其它用户无法正常访问，这是不可接受的。

但是这不意味着这个算法没用，很多时候我们的需求就是很简单，只需要控制并发数，如果不考虑上述情况的话是完全可用的。

## 2.滑动窗口计数法
这个算法实际上是前面这种算法的改进版，第一种方法的问题在于统计的时间太粗了，不够精细，假设我们把1s细分为1000ms，那么也就是10ms一个请求，这时候我们只需要统计最近100个10ms段内的请求数就可以了，而不是一个固定的开始时间。这个算法有一个非常知名的应用就是在TCP网络中流量控制中用于计算窗口大小。

<img src="/images/2020-06-16_22-52.png" />

滑动窗口的核心思想是把时间段切割成更小的区块，每个区块单独计数，最后统计这个时间段内各个小区块的总计数，其重点在于这个时间段并不是一个固定的时间段，而是在动态变化。

为了便于演示，这里假设限流条件为10s内最多500个请求，我们以1s为区块划分为10个区块，每个区块单独计数，这个算法实现的核心在于维持一个队列，这里使用了Go自带的list实现，参考代码如下：
```go
package main

import (
    "container/list"
    "fmt"
    "time"
)

type part struct {
    Time int64
    Count int
}

func main() {
    var listPart = list.New()
    for {
        var nowSecond = time.Now().Unix()
        if listPart.Len() == 0 {
            for i := 0; i < 10; i++ {
                listPart.PushBack(part{
                    Time: nowSecond+int64(i),
                    Count: 0,
                })
            }
        }

        // 打印方便调试
        for e := listPart.Front(); e != nil; e = e.Next() {
            fmt.Printf("e = %v\n", e.Value)
        }

        if listSum(listPart) >= 500 {
            // 右移动队列元素
            if listPart.Back().Value.(part).Time == nowSecond{
                listPart.Remove(listPart.Front())
                listPart.PushBack(part{
                    Time: nowSecond+1,
                    Count: 0,
                })
            }else {
                fmt.Println("reach limit")
                time.Sleep(time.Second)
                continue
            }
        }

        time.Sleep(time.Millisecond*10)
        doSomething()

        element := listGet(listPart, nowSecond)
        count := element.Value.(part).Count
        element.Value = part{
            Time: nowSecond,
            Count: count+1,
        }
    }
}

func doSomething() {
    fmt.Printf("do something \n")
}

// 队列求和
func listSum(l *list.List) int {
    var sum int
    for e := l.Front(); e != nil; e = e.Next() {
        sum += e.Value.(part).Count
    }
    return sum
}

// 查找队列元素
func listGet(l *list.List, t int64) *list.Element {
    for e := l.Front(); e != nil; e = e.Next() {
        if e.Value.(part).Time == t {
            return e
        }
    }
    return nil
}
```
实际运行中，整个队列元素保持不变，一直是10个，但是其时间段一直在变，比如下面这个情况下，前面所有的时间区块计数之和已经达到500，开始限流。
```
e = {1592323440 53} //即将被移除
------------------------------------
e = {1592323441 98}
e = {1592323442 98}
e = {1592323443 98}
e = {1592323444 98}
e = {1592323445 55}
e = {1592323446 0}
e = {1592323447 0}
e = {1592323448 0}
e = {1592323449 0}
-------------------------------------
e = {1592323450 0} //即将新入列
-------------------------------------
```
随着时间的推进，新加入一个区块，就有新的流量可用了，如此往复不间断，实现了动态时间窗口！

这种实现方法区块划分的越多越精确,但是到底该设置多少个格子才足够精确呢？而且这种方式依然没有解决突发流量瞬间占用的情况，所以又延伸出了2种流行的平滑限流算法分别是```漏桶算法```和```令牌桶算法```。

