---
title: 浅谈Golang反射及其应用
date: 2020-08-16 15:00:00
tags: Golang
category: Golang
---
说到反射机制，很多语言都有，通常来说，反射可以在运行中获取数据对象信息，它提供了一种在运行过程中动态修改代码自身的能力，也被认为是元编程的一种实现方式。

通过反射我们可以实现一些正常情况下无法实现的功能，甚至简化代码，提高工作效率。虽然反射并不是必须的，但是用的好确实可以事半功倍，下面咱们就看看Go里面反射的一些应用。

<!--more-->
## 1.JSON序列化
下面这段代码很常见，可以把一个Go的结构体对象转换成json格式：
```go
package main

import (
    "encoding/json"
    "fmt"
)

type Item struct {
    Name  string `json:"name"`
    Price int    `json:"price"`
}

func main() {
    i := Item{
        Name:  "json",
        Price: 10,
    }
    res, _ := json.Marshal(i)
    fmt.Printf("%s\n", res)
}
//结果： {"name":"json","price":10}
```
大家首先得理解json这种格式，json是一种数据交换格式，目前主要用于http接口，其特点是简单易懂，省流量。因为是一种数据交换格式，所以我们可以说json是跨语言的，无论你是使用PHP还是Java，都会接触到json。

很多人不知道json的字段也是有类型的，只不过其类型更简单，常见的数据类型有number、字符串，比如number对应的是Go里面的int和float。所以，在序列化的时候问题来了，我们需要把Go里面类型转换成json里面对应的类型，在这个过程中我们就需要使用反射获取当前数据对象的类型，然后做一个映射关系。

```go
func (e *encodeState) marshal(v interface{}, opts encOpts) (err error) {
    defer func() {
        if r := recover(); r != nil {
            if je, ok := r.(jsonError); ok {
                err = je.error
            } else {
                panic(r)
            }
        }
    }()
    e.reflectValue(reflect.ValueOf(v), opts)
    return nil
}
```
除了获取对象的类型之外，还有一个就是获取tag信息，比如json，这样我们就可以在序列化的时候指定字段名称，当然我们也可以自定义其它tag，用作其它用途。

## 2.反射的常见操作
Go里面有一个reflect包用于反射操作，最常见的用法莫过于 TypeOf 和 ValueOf这2个方法，通过这2个方法，我们可以非常轻松的获取对象的类型和属性：
```go
func main() {
    i := Item{
        Name:  "json",
        Price: 10,
    }
    t := reflect.TypeOf(i)
    v := reflect.ValueOf(i)

    fmt.Printf("%v\n", t)
    fmt.Printf("%v\n", v)
}
```
通过ValueOf可以拿到反射的对象，然后能做的事情就非常多了，比如说对象的成员、对象实现的方法，然后可以获取访问、修改其成员值，也可以调用其方法，举个例子：
```go
func (r Item) Say() string {

    fmt.Printf("name = %s\n", r.Name)

    return r.Name
}
```
我们先给一个struct加个方法，然后使用反射去调用：
```go
func main() {
    i := Item{
        Name:  "json",
        Price: 10,
    }
    v := reflect.ValueOf(i)
    // 第1个成员值
    fmt.Printf("%v\n", v.Field(0))
    // 第2个成员值
    fmt.Printf("%v\n", v.FieldByName("Price"))
    // 调用其方法
    fmt.Printf("%v\n", v.Method(0).Call(nil))

    //修改，注意这里传的是地址
    m := reflect.ValueOf(&i)
    m.Elem().Field(0).SetString("JSON")
    m.Elem().FieldByName("Price").SetInt(100)
    fmt.Printf("%v\n", i.Name)
    fmt.Printf("%v\n", i.Price)
}
```

利用反射我们还可以实现泛型，简化操作，比如下面这个方法：
```go
func main() {
    fmt.Printf("%v\n", Add(1, 2))     //3
    fmt.Printf("%v\n", Add(1.1, 2.4)) //3.5
}

func Add(a, b interface{}) interface{} {
    switch reflect.TypeOf(a).String() {
    case "int":
        return reflect.ValueOf(a).Int() + reflect.ValueOf(b).Int()
    case "float64":
        return reflect.ValueOf(a).Float() + reflect.ValueOf(b).Float()
    }
    return nil
}
```
## 3.结构体tag信息
获取结构体Tag信息的过程稍微麻烦一点，ValueOf的参数必须是指针类型，然后迂回调了几个函数才能获取到，细心的人估计注意到了，修改成员的值的时候也必须传指针。
```go
func main() {
    i := Item{
        Name:  "json",
        Price: 10,
    }
    m := reflect.ValueOf(&i)

    fmt.Printf("%v\n", m.Type().Elem().Field(0).Tag.Get("json"))
}
```

总结的说，Go的反射目前应用主要有2大方面，一方面是做不同数据类型系统之间的转换操作，比如json，其实序列化的工作就是把json类型和Go类型做一个转换。另外，一些数据库的ORM，我们也要把数据库的数据类型转换成Go类型。还有比如一些配置文件的解析，json、ini、yaml等都需要利用反射实现。另一个方面就是一些复杂编程的实现，比如依赖注入框架，可以简化用户编码操作，实现一些动态、灵活的系统。