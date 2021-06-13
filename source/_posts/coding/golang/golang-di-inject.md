---
title: Golang的依赖注入之inject库
date: 2021-06-10 22:09:49
tags: Golang
category: Golang
---

之前写过一篇 [文章](https://wangbjun.site/2019/coding/golang/golang-di.html) 简单介绍过Go里面的依赖注入，其中提到了Facebook开源的一个依赖注入项目，地址：https://github.com/facebookgo/inject 刚好最近看一个开源项目用到这个项目，于是带着一些疑问仔细研究了一下。

<!--more-->

## 1.简介
这个包提供了一个基于反射的注入器，它默认使用单例，支持可选的私有实例以及命名实例。

例如：
```
`inject:""`
`inject:"private"`
`inject:"dev logger"`
```
第一个无值语法是针对关联类型的单例依赖的常见情况。第二个触发器创建关联类型的私有实例。最后一个是要求一个名为 “dev logger” 的依赖关系。

## 2.案例
假设我们现在有2个对象，其中对象App依赖Logger的log方法，按照传统的写法，我们需要先实例化Logger对象，然后把其作为参数传入App实例里面。

但是使用inject库的话，我们同样可以实现这个效果，更加简洁。
```go
type Logger struct {
    cfg string
}

func (Logger) log() {
    log.Printf("write log")
}

type App struct {
    Logger Logger `inject:""` //依赖logger
}

func (r App) run() {
    r.Logger.log()
}

func main() {
    //传统写法
    app := App{
        Logger: Logger{},
    }
    app.run()

    //依赖注入写法
    app2 := App{}
    var g inject.Graph
    _ = g.Provide(&inject.Object{Value: &app2},&inject.Object{Value: &Logger{}})
    _ = g.Populate()
    app2.run()
}
```

## 3.基本原理
这个库使用起来非常简单，主要分为3步，第一步，在需要注入依赖的结构体里面使用inject的标签标注；第二步，初始化一个graph对象，这个graph意思是图表；第三步，调用provide方法添加所有依赖的对象，然后使用populate方法完成注入。

先不看代码，说说注入的思想，我们要想解决依赖，首先肯定得把依赖创建好放在某个地方，然后通过某种方式构建依赖关系，或者说当某个对象需要某个依赖的时候我们自动满足其需求。不同语言的实现方法也不一样，比如在PHP里面是单进程的运行方法，所有的对象都是全局的，所以实现起来相对简单。

这个库里面graph就相当于一张空白的地图，其结构体如下：
```go
// The Graph of Objects.
type Graph struct {
    Logger      Logger // Optional, will trigger debug logging.
    unnamed     []*Object
    unnamedType map[reflect.Type]bool
    named       map[string]*Object
}
```
然后是图里面的对象的结构：
```go
// An Object in the Graph.
type Object struct {
    Value        interface{}
    Name         string             // Optional
    Complete     bool               // If true, the Value will be considered complete
    Fields       map[string]*Object // Populated with the field names that were injected and their corresponding *Object.
    reflectType  reflect.Type
    reflectValue reflect.Value
    private      bool // If true, the Value will not be used and will only be populated
    created      bool // If true, the Object was created by us
    embedded     bool // If true, the Object is an embedded struct provided internally
}
```

Provide方法的逻辑比较简单，做了一些判断，最终还是把对象放到unamed和named里面存起来了，而且Populate这个方法就是按照你Provide的顺序挨个对这些对象解析，然后通过反射的机制把需要的对象给注入，其中实现过程和逻辑还比较复杂。

## 4.开源项目应用
在一个知名的go开源项目```Grafana```里面就使用到了inject库，在这个项目里面，有很多service服务，作者通过init函数注册服务，然后在server启动的时候初始化所有的服务，同时解决各个服务之间的依赖关系，感兴趣的可以看一看源码，写的非常好。

```go
func init() {
    registry.RegisterService(&ShortURLService{})
}

type Service interface {
    GetShortURLByUID(ctx context.Context, user *models.SignedInUser, uid string) (*models.ShortUrl, error)
    CreateShortURL(ctx context.Context, user *models.SignedInUser, path string) (*models.ShortUrl, error)
    UpdateLastSeenAt(ctx context.Context, shortURL *models.ShortUrl) error
    DeleteStaleShortURLs(ctx context.Context, cmd *models.DeleteShortUrlCommand) error
}

type ShortURLService struct {
    SQLStore *sqlstore.SQLStore `inject:""`
}

func (s *ShortURLService) Init() error {
    return nil
}
```
## 5.性能问题
说到反射，很多人第一感觉就是会不会比较慢，go的反射性能差是公认的，要说有多慢，大概也就是100多倍，换算到实际应用，也就是5ms和500ms的区别，如果说你的应用对性能追求不是很极致，用一用问题不大。

但是，也看你咋用，在```Grafana```这个项目里面，只有在服务首次运行的时候才会做注入操作，也就是说，即使慢也只有启动的时候慢，在整个服务初始化完成之后，后面的操作就不需要用到反射了。

所以说，这种用法对性能几乎没有太大影响。

不过在这个项目里面，作者还使用反射机制实现了一个Bus事件机制，在每次```Disptach```的时候都会使用到反射获取调用的对象和参数：
```go
// Dispatch function dispatch a message to the bus.
func (b *InProcBus) Dispatch(msg Msg) error {
    var msgName = reflect.TypeOf(msg).Elem().Name()

    withCtx := true
    handler := b.handlersWithCtx[msgName]
    if handler == nil {
        withCtx = false
        handler = b.handlers[msgName]
        if handler == nil {
            return ErrHandlerNotFound
        }
    }

    var params = []reflect.Value{}
    if withCtx {
        params = append(params, reflect.ValueOf(context.Background()))
    }
    params = append(params, reflect.ValueOf(msg))

    ret := reflect.ValueOf(handler).Call(params)
    err := ret[0].Interface()
    if err == nil {
        return nil
    }
    return err.(error)
}
```
虽然在这个项目里面大量使用了反射机制，我在使用这个项目的时候并没有感觉有多慢，可能这并不是一个对性能有极致追求的项目，并发也不高。