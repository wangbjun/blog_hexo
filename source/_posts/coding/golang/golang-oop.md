---
title: Go面向对象写法
date: 2019-04-19 22:05:43
tags: Golang
category: 编程开发
---

Go并不是一个类似于Java、C++，或PHP这样内置面向对象语法的操作的语言，在Go里面名义上是没有类（class）这个概念的，但是这并不代表Go不能面向对象，毕竟面向对象只是一种设计思想！

为什么Go并不原生支持面向对象呢？这是一个问题

接下来，我会从面向对象的三大特性封装、继承、多态这个几个方面来讲讲Go是怎么实现的OOP的。

<!--more-->

### 1.封装
闲话少说，在Go里面可以使用结构体模拟类:
```
type Goods struct {
    name  string
    price int
}
```
在Go里面有一个约定俗称的规则，变量名、结构体名、结构体属性成员名大写代表是公开权限，可以被其它包使用。类似于类的**public**属性。如果小写就类似于**private**属性。

类里面除了属性之外，一般会有自己的方法，在Go里面可以这样实现(这里我采用的是Go modules结构)：
```
package models

import "fmt"

type Goods struct {
    Name  string
    Price int
}

func (g *Goods) GetName() string {
    return g.Name
}

func (g *Goods) SetName(name string) {
    g.Name = name
}

func (*Goods) String() {
    fmt.Println("I am Goods")
}
```
其实就是在函数名前加一个类型声明，如果你在方法里面不需要使用类本身，则可以省略参数标识。

如何使用这个“类呢”？
```
package main

import (
    "demo/models"
    "fmt"
)

func main() {
    goods := models.Goods{
        "笔记本", 100,
    }
    
    fmt.Printf("Goods name is %s\n", goods.GetName())
    
    goods.SetName("小米笔记本")
    
    fmt.Printf("Goods name is %s\n", goods.GetName())
}
```

我们可以采用字面量赋值的方式初始化对象，虽然结构体并没有构造函数这个东西，但是我们可以造个差不多的方式出来。

新增这个方法：
```
func NewGoods(name string, price int) Goods {
    g := Goods{
        Name:  name,
        Price: price,
    }
    return g
}
```
然后我们就可以这样使用：
```
var goods models.Goods
goods = models.NewGoods("笔记本", 1000)
```
其实区别倒是不大，封装了一下，更加简洁，虽然达不到构造函数自动调用的效果。

### 2.继承
Go里面并没有extends这样的语法，但是结构体的成员可以是结构体，这实际上是使用组合实现了继承的效果。
```
package models

type Apple struct {
    Goods //继承了Goods
    Color string
}

// 构造函数
func NewApple(name string, price int, color string) Apple {
    apple := Apple{
        Goods{name, price},
        color,
    }
    
    return apple
}
```
main.go:
```
package main

import (
    "demo/models"
    "fmt"
)

func main() {
    apple := models.NewApple("红富士苹果", 200, "red")
    fmt.Printf("Apple name is %s", apple.GetName())
}
```
Apple可以使用Goods的方法和属性，使用组合的好处就是不存在多继承的限制，在很多面向对象的语言里面，只能单继承。

### 3.多态
虽然Go里面也没有implements这样的关键字，但是在Go里面可以使用interface来实现多态效果，而且Go里面的接口相当灵活。

定义接口：
```
package models

type Saleable interface {
    Sell()
}
```
实现接口(Apple)：
```
func (Apple) Sell()  {
    fmt.Println("我实现了saleable接口")
}
```
使用：
```
func main() {
    apple := models.NewApple("红富士苹果", 200, "red")
    
    var i models.Saleable
    
    i = &apple
    
    i.Sell()
}
---结果---
我实现了saleable接口
```
划重点，在GO里面只要一个结构体（struct）定义了一个接口(interface)里面的所有方法，就意味着这个这个struct实现了这个接口，这是隐式的。可见，在Go里面接口还是挺好用的。

