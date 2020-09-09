---
title: Golang函数参数值传递|引用传递
date: 2020-08-28 22:30:00
tags: Golang
category: Golang
---
在很多语言里面，函数传参分为值传递和引用传递，在说明这2者区别之前咱先回顾一下函数的一些概念，函数参数有形参和实参之分，所谓形参就是形式参数，它是指定义函数时候的声明的参数，比如在Golang里面 **func do(a string, b int)**，a和b就是形参。而实参则是你在调用这个函数时候所传递的参数，比如在下面代码里面，pa和pb就是实参。
```go
func main() {
    var (
        pa = "1"
        pb = 2
    )
    do(pa, pb)
}

func do(a string, b int) {
    fmt.Printf("%s --- %d", a, b)
}
```
其中，值传递指的是在函数调用的时候，内存会复制一份实参的值给形参，换句话说，形参和实参互不影响！而引用传递则是则形参和实参指向同一块内存区域，换句话，你修改形参也会影响实参的值，修改实参也会影响形参的值！

<!--more-->

网上很多文章都说到Golang里面函数参数默认为 **值传递**，但是很多时候并不是你想象的那么简单，不信咱们先看一段代码：
```go
func main() {
    p := []string{"1", "2"}
    go do(p)
    p = append(p, "3")

    m := make(map[string]string)
    m["1"] = "1"
    m["2"] = "2"
    go doMap(m)
    m["3"] = "3"

    time.Sleep(time.Second * 3)
}

func do(a []string) {
    time.Sleep(time.Second)
    fmt.Printf("%v\n", a)
}

func doMap(a map[string]string) {
    time.Sleep(time.Second)
    fmt.Printf("%v\n", a)
}
// 运行结果
[1 2]
map[1:1 2:2 3:3]
```
这2个函数的区别之处就在于一个形参是slice，一个是map类型，但是从运行结果可以得知，slice没有修改成功，而map则成功修改了，似乎map的传参方式是引用传递？？？

严格来说，大部分人文章说到Golang参数默认是值传递是没问题的，对于基本数据类型也就是整型、浮点型、字符串、数组以及slice等类型默认就是值传递，但是如果你在参数面前加个指针的话，**它并不会变成引用传递**，它依然是值传递，只不过传递的是指针地址，虽然看上去和引用传递效果是一样的，比如：
```go
func main() {
    p := make([]string, 0)
    p = append(p, "1")
    p = append(p, "2")

    fmt.Printf("%p\n", p) // 0xc00000c060

    do(&p)

    fmt.Printf("%v\n", p)
}

func do(a *[]string) {
    fmt.Printf("%p\n", *a) // 0xc00000c060
    *a = append(*a, "3")
}
```
对于map来说，之所以会出现开始例子里面那种情况是因为 make map返回是一个指针地址，所以它传递也是指针地址。这一点可以通过查看 src/runtime/hashmap.go 源代码发现，make函数返回的是一个hmap类型的指针 *hmap。其实这一点对于chan类型也是一样的，chan默认也是指针传递。

对于结构体来说，默认也是值传递，如果你想在被调用函数内部修改结构体的值，也需要传指针地址才行。
```go
type Some struct {
    name string
}

func main() {
    s := Some{
        name: "1",
    }

    // 传递值
    do(s)

    fmt.Printf("%v\n", s) // {"1"}

    // 传递指针
    doPointer(&s)

    fmt.Printf("%v\n", s) // {"2"}
}

func do(a Some) {
    a.name = "2"
}

func doPointer(a *Some) {
    a.name = "2"
}
```

最后，什么时候该使用引用传递（传指针地址）呢？

一般来说，有2种应用场景，第一种则是上面演示的代码场景，即我们需要在函数内部修改传入的参数值，这种需求其实挺常见。

另一种则是为了节省内存，提高程序运行效率，因为值传递需要在内存里面copy赋值一份值给形参，如果实参数据量特别大，这个过程不仅慢而且也需要消耗额外的内存，这时候使用指针参数最合适，只需要copy指针就行。
```go
func main() {
    p := []string{"1", "2", "3"}
    doPointer(&p)
}

func doPointer(p *[]string) {
    a := append(*p, "999")
    fmt.Printf("%v\n", a)
}
```

根据个人编程经验来说，如果你没有上述需求，那么只需注意一个map和chan类型的特殊情况以避免出现意外，不过除此之外，使用指针还有一个好处就是指针可以是nil，所以在返回错误值的时候可以直接return nil，比较简洁省事。
```go
func do() Some {
    return Some{}
}

func doPointer() *Some {
    return nil // 省事
}
```