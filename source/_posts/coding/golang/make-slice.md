---
title: Golang里面slice初始化的坑
date: 2020-06-06 23:42:22
tags: 
- Golang
- Slice
category: Golang
---
相信很多人对Golang里面的数组都不陌生，但实际上99%的场景我们使用的都是slice，原因很简单，Go里面的数组类似C数组长度是固定的，局限太多，而slice则是一个变长的数组，可以自动扩容，类似JS、PHP等弱类型语言里面的数组。

但实际使用slice的过程中，我们一般会遇到2种写法，下面咱们就说说这2种方式的差异和存在的坑：
```go
var s []string

var s = make([]string, 0)
```
<!--more-->

## 1.make和new区别
首先，咱先说说make的使用，make是Go的内置函数，专门用于分配和初始化指定大小的slice、map、chan类型，它返回的是一个type。而new则不同，new返回的是一个*type,也就是一个指针类型，指向type的零值。

在使用make初始化slice的时候，其第二个参数是slice的长度length（必填，可为0），第三个参数是容量capacity（选填），new的话只有一个参数。

比如下面这些写法，前面2种大部分情况下是等价的，使用起来并没有多大区别，但是第三种new的写法就稍微有点区别，因为它的返回值是指针。
```go
var s []string
s = append(s, "1")
s = append(s, "2")

var ss = make([]string, 0)
ss = append(ss, "1")
ss = append(ss, "2")

var sss = new([]string)
*sss = append(*sss, "1")
*sss = append(*sss, "2")

fmt.Printf("%v\n", s)
fmt.Printf("%v\n", ss)
fmt.Printf("%v\n", sss)

//结果
[1 2]
[1 2]
&[1 2]
```
实际应用中，slice、map、chan必须使用make初始化，new则很少用，偶尔用于结构体的初始化，但是一般结构体我们会采用更加简单的字面量声明方式。

## 2.make指定slice大小和容量
```go
var s []string
```
上面这种初始化方式，length和capacity默认是为0
```go
var s = make([]string, 3, 10)
```
而上面这种方式，则是初始化了一个长度为3，容量为10的slice，也就是说里面已经有3个元素了，但是值是这些元素类型的零值，对于string来说就是空字符串。

实际应用中，如果你对接下来使用的容量有一个预计，则可以提前开辟好内存空间，避免slice后期自动扩容，毕竟扩容也有性能开销。

## 3.JSON序列化
其实这点是比较奇怪的地方，也是差异最大的地方，在拿Go写API接口的时候，我们经常需要把结果序列化成JSON返回。

```go
var s []string
if true {
    s = append(s, "1")
    s = append(s, "2")
    s = append(s, "3")
    s = append(s, "4")
}
result, err := json.Marshal(s)
if err != nil {
    panic(err)
}
fmt.Printf("%s\n", result)

//结果
["1","2","3","4"]
```
比如上面这个例子，正常情况下是没问题，但是如果 if 的条件未成立，s则是一个空的slice，结果就不一样了，返回的是null，这个就不太好了，对于前端来说，空数组应该是```[]```而不是null，从接口规范来说，我们应该保持返回类型一致。

但是如果你使用make初始化则不存在这个问题：
```go
var s = make([]string, 0)
result, err := json.Marshal(s)
if err != nil {
    panic(err)
}
fmt.Printf("%s\n", result)
//结果
[]
```
这是为什么呢？

<b>在Go里面，当一个变量被声明了但是没有初始化值的话，其默认值则是该类型的零值，比如string默认零值是空字符串，int默认是0，对于slice来说其零值是nil。</b>所以严格来说，```var s []string```申明的是一个nil slice，而```make```初始化的是一个空slice，它们俩是有区别的，看一下下面的代码：
```go
var s []string
fmt.Printf("%v\n", s == nil) //true

var ss = make([]string, 0)
fmt.Printf("%v\n", ss == nil) //false
```
虽然说大部分情况下使用起来都没有区别，但是在json序列化的时候，nil直接就被处理成了null。。。


最后总结，建议大家使用make初始化slice，同时也不建议通过判断slice是否为nil去处理一些逻辑，建议更加靠谱的方式，比如slice的length。
