---
title: Web框架Gin的封装处理
date: 2020-07-01 22:40:10
tags: 
- Golang
- Gin
category: Golang
---
众所周知，Golang非常适合用于开发高性能的API，开源的Web框架也很多，比如国产的Beego，以及在目前最流行的Gin，还有Echo、Iris、Revel等框架。

由于博主之前从事过PHP，相比来说，感觉Go的这些框架都比较轻量，很多时候还需要自己花点功夫再打理一下，比如这些框架都没有处理数据库这块，Beego虽然自带了一个ORM，但是可用性极差，需要我们自己手动去集成一些开源的组件。

就拿我用过比较多的Gin框架（ https://github.com/gin-gonic/gin ）来说，它属于其中最轻量级的框架，主要功能包括路由、请求参数、中间件、模板等，存在一些缺失的东西，比如：

* 没有推荐项目结构模块划分，很多人不知道代码咋存放

* 缺少数据库ORM，虽说Go标准库支持数据库，但是缺少封装，用起来麻烦

* 缺少配置文件加载功能，任何项目都少不了配置文件

* 日志处理功能不够强大

所以，我基于Gin加上一些开源的组件，封装了一个不算是框架的框架，这里暂且称其为Gen，主要是便于快速开发，解决一些通用性的问题。

地址：https://github.com/wangbjun/gen

<!--more-->

## 1.项目结构
这里参考借鉴了PHP的Web框架 **Laravel** 的结构，整体上是一个MVC结构：

<img src="/images/2020-07-01_22-15.png" width="50%"/>

基本上包含一个Web框架应该有的东西，包括路由、控制器层、模型层、service层等，便于我们存放代码，开发起来也方便很多，通过文件夹名字我们就能了解各个包的主要作用。

项目使用Go Mod解决依赖，接下来我依次介绍一些主要模块的功能

## 2.Main入口
```go
package main

import (
    "gen/config"
    "gen/router"
    "github.com/gin-gonic/gin"
    "log"
)

func main() {
    gin.SetMode(getMode())
    engine := gin.New()
    engine.Use(gin.Recovery())
    // 加载路由
    router.Route(engine)
    // 启动服务器
    log.Println("server started success")
    err := engine.Run(":" + config.GetAPP("PORT").String())
    if err != nil {
        log.Fatalf("server start failed, error: %s", err.Error())
    }
}

func getMode() string {
    debug := config.GetAPP("DEBUG").String()
    if debug == "true" {
        return gin.DebugMode
    }
    return gin.ReleaseMode
}
```
main里面第一步就是加载配置文件，然后根据配置文件设置Gin的模式以及服务监听的端口，然后这里还有一些”隐藏“的init初始化操作，比如在初始化路由的过程中加载了数据库配置并且建立数据库连接、还有初始化日志配置等，具体可以查看各个包的init函数。

## 3.配置文件
默认配置文件是同级目录下的 **app.ini**，这里采用了```gopkg.in/ini.v1```开源库解析配置，格式也是非常简单的k-v格式，举个例子：
```
[APP]
PORT = 8080
DEBUG = true
URL = http://127.0.0.1:8080
LOG_FILE = storage/logs/app.log
LOG_LEVEL = info

[DB]
Dialect = mysql
DSN = root:123456@tcp(127.0.0.1:3306)/blog?charset=utf8mb4&parseTime=True&loc=Local
MAX_IDLE_CONN = 5
MAX_OPEN_CONN = 50
```
加载配置文件的代码位于```config/Config.go```文件里面，逻辑非常简单，也支持通过”-c“指定配置文件，这里定义了一个包全局变量，一次加载，终身使用，具体的日志读取API可以参考这个库的官方文档。

## 4.日志处理
日志这块采用了来自uber的 **zap** 库，高性能，扩展性也很强，代码位于```zlog/ZapLogger.go```文件，主要操作是根据加载的配置文件初始化日志级别、存储位置、存储格式以及自动分割等配置。

默认情况下，日志存储在```storage/logs```文件夹下，格式是JSON

日志这块有一个特殊的功能，可以选择为每个日志加上traceId，便于追踪排查问题，核心代码如下：
```go
func getContext(ctx *gin.Context) []zap.Field {
    var (
        now          = time.Now().Format("2006-01-02 15:04:05.000")
        processId    = os.Getpid()
        startTime, _ = ctx.Get("startTime")
        duration     = float64(time.Now().Sub(startTime.(time.Time)).Nanoseconds()/1e4) / 100.0 //单位毫秒,保留2位小数
        serviceStart = startTime.(time.Time).Format("2006-01-02 15:04:05.000")
        request      = ctx.Request.RequestURI
        hostAddress  = ctx.Request.Host
        clientIp     = ctx.ClientIP()
        traceId      = ctx.GetString("traceId")
        parentId     = ctx.GetString("parentId")
        params       = ctx.Request.PostForm
    )
    return []zap.Field{
        zap.String("traceId", traceId),
        zap.String("serviceStart", serviceStart),
        zap.String("serviceEnd", now),
        zap.Int("processId", processId),
        zap.String("request", request),
        zap.String("params", params.Encode()),
        zap.String("hostAddress", hostAddress),
        zap.String("clientIp", clientIp),
        zap.String("parentId", parentId),
        zap.Float64("duration", duration)}
}
```
如果想记录当前请求的一些信息可以使用```zlog.WithContext(ctx).Sugar().Infof("log msg")```这种写法来记录，同理，详细API可以参考其官方文档。

## 5.数据库处理
这里采用了知名的**gorm**开源库，数据库的配置在```Conf/Database.go```里面，这里可以定义多个数据库配置：
```go
package config

var DBConfig map[string]map[string]string

func init() {
    DBConfig = map[string]map[string]string{
        "default": {
            "dialect":      Conf.Section("DB").Key("Dialect").String(),
            "dsn":          Conf.Section("DB").Key("DSN").String(),
            "maxIdleConns": Conf.Section("DB").Key("MAX_IDLE_CONN").String(),
            "maxOpenConns": Conf.Section("DB").Key("MAX_OPEN_CONN").String(),
        },
        "user": {
            "dialect":      Conf.Section("DB").Key("Dialect").String(),
            "dsn":          Conf.Section("DB").Key("DSN").String(),
            "maxIdleConns": Conf.Section("DB").Key("MAX_IDLE_CONN").String(),
            "maxOpenConns": Conf.Section("DB").Key("MAX_OPEN_CONN").String(),
        },
    }
}
```
然后在```model/DB.go```文件里面，初始化了所有的DB配置，并且提供一个快速访问的函数：
```go
// 获取默认db
func DB() *gorm.DB {
    conn, ok := dbConnections["default"]
    if !ok {
        return nil
    }
    return conn
}

// 获取user db
func UserDB() *gorm.DB {
    conn, ok := dbConnections["user"]
    if !ok {
        return nil
    }
    return conn
}

// 获取db连接
func GetDB(name string) *gorm.DB {
    conn, ok := dbConnections[name]
    if !ok {
        return nil
    }
    return conn
}
```
关于gorm的详细用法可以参考其官方文档：https://gorm.io/
## 6.其它
middleware文件夹里面主要一些中间件，比如说用户鉴权、请求日志的中间件，使用的时候需要在路由里面配置，这个可以参考Gin框架文档。

目前我的习惯是**MVC+Service**这套结构（纯api的话就没有V了），不过如果业务逻辑简单，直接全写在控制器里面也没多大问题。但是如果使用这套结构一定要注意划分好代码层次，不要出现相互引用的情况，比如说model包使用controller包定义的东西，然后controller又使用了model包里面的东西，这样的写法在PHP里面没问题，但是在Go里面无法通过编译。

所以，从最佳实践上说，应该是controller调model，model调service这个顺序，不可反向操作。

最后，这个项目自带了一个包括用户注册登录以及发表文章、查看文章的API的功能，具体可以查看项目README文件，如果想使用的话建议直接clone本项目，然后在这个基础上修改。

