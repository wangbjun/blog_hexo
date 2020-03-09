---
title: Golang的依赖注入简介
date: 2019-05-15 21:13:49
tags: Golang
category: Golang
---

DI - Dependency Injection，即“依赖注入”，是指组件之间依赖关系由容器在运行期决定，与此同时还有一个叫作IOC的词汇，IOC即控制反转。

理论上讲，这2个概念都是基于OOP编程而产生的思想，在OOP编程里面，我们强调对象之间的依赖关系，比如说对象B依赖对象A的某些功能，我们就说B依赖A。

DI毕竟不是Go语言的专利，它是一种编程思想，在很多语言里面都有体现和实现，相信很多具有编程经验的人也有所了解，下面咱们直接开始讲在Go语言里面怎么使用DI。

<!--more-->

## 实现方式
Golang的DI目前主要有2种方式，一种是使用反射特性实现，代表开源项目有facebook/inject，还有uber/dig。另一种是代码自动生成，代表开源项目有google/wire。

下面咱们看一个案例：

由于Go并不是纯OOP语言，所以这里使用struct模拟对象的概念，有3个对象，其中App依赖DB和Redis。

DB：
```go
package Object

import "fmt"

type DB struct {
    config string
}

func (DB) Get() {
    fmt.Println("I am DB")
}
```

Redis：
```go
package Object

import "fmt"

type Redis struct{}

func (Redis) Get() {
    fmt.Println("I am Redis")
}
```
App:
```go
package Object

import "fmt"

type App struct {
    r  Redis
    db DB
}

func (p App) Work() {
    fmt.Println("I can work")
    p.db.Get()
    p.r.Get()
}
```
如果不使用依赖注入，我们只能手动解决依赖，代码如下：
```go
package main

import "di/Object"

func main() {
    app := Object.App{
        R:  Object.Redis{},
        DB: Object.DB{},
    }
    app.Work()
}
```
这种写法并无太大问题，简单安全，不过项目非常大的时候，对象之间依赖关系复杂，手动解决依赖可能非常麻烦，这时候就需要自动注入依赖了。


## facebook/inject

这是Facebook开源的一个项目，地址：github.com/facebookgo/inject

它使用struct的tag声明依赖，第一个无值语法是针对关联类型的单例依赖的常见情况。第二个触发器创建关联类型的私有实例。最后一个是要求一个名为 “dev logger” 的依赖关系。
```
`inject:""`
`inject:"private"`
`inject:"dev logger"`
```

下面以App为例：
```go
type App struct {
    R  Redis `inject:""`
    DB DB `inject:""`
}
```
运行：
```go
package main

import (
    "di/Object"
    "github.com/facebookgo/inject"
)

func main() {
    var g inject.Graph

    var app Object.App

    _ = g.Provide(
        &inject.Object{Value: &Object.DB{},},
        &inject.Object{Value: &Object.Redis{},},
        &inject.Object{Value: &app,
        })

    _ = g.Populate()

    app.Work()
}
```
给struct tag只是第一步，在程序启动的时候需要先注入依赖。

### 性能问题
一般说到依赖注入必然会用到反射，说到Go的反射，大多数人都会说性能很差。

这个inject库也是用到了反射原理，性能会不会很差呢？

其实还是看用法，官方推荐在应用程序启动的时候注入所有依赖，而不是在运行中注入依赖，这样即使慢，也只是程序每次启动的时候慢，并不影响后续的运行情况。
