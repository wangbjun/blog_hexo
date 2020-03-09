---
title: 解决Golang测试配置文件加载问题
date: 2019-11-19 11:49:25
tags: Golang
category: Golang
---

最近在写Go的项目，使用的框架是Gin，众所周知，Gin是一个比较简单的框架，只提供了核心功能，并没有配置文件模块，所以这块得自己搞了，Go的第三方解析配置的库非常多，无论是ini、yaml、json文件支持都非常好，而且Go的项目一般都是常驻进程的，所以只需要在项目启动的时候解析一次就行可以了。

## 示例
最简单的办法通常就是定义一个全局的配置变量供其它包使用，在init函数里面初始化加载配置文件，示例如下：

<!--more-->

```go
package config

import (
    "gopkg.in/ini.v1"
    "log"
    "os"
)

var Conf Config

type Config struct {
    App      App
}

type App struct {
    Port    string
    Debug   string
    Url     string
    LogFile string
}

func init() {
    envFile := "app.ini"
    conf, err := ini.Load(envFile)
    if err != nil {
        log.Panicf("parse conf file [%s] failed, err: %s", envFile, err.Error())
    }
    sectionApp := conf.Section("APP")
    Conf.App = App{
        Port:    sectionApp.Key("PORT").String(),
        Debug:   sectionApp.Key("DEBUG").String(),
        Url:     sectionApp.Key("URL").String(),
        LogFile: sectionApp.Key("LOG_FILE").String(),
    }
    log.Println("init config file success")
}
```
默认情况下，入口文件main.go文件都是位于项目根目录下面，和app.ini文件同级，所以这种写法完全没问题。

## 问题
但是当你跑测试用例的时候，而且当这个测试用例并不在项目根目录的时候就会产生问题: 找不到配置文件。

原因很简单，Go的测试用例最佳实践是和被测试的文件放在一起，所以测试文件可能在二级、三级甚至多级目录里面，如下图：
```bash
├── app.ini
├── config
│   ├── Config.go
│   └── DataBase.go
├── controller
│   ├── BaseController.go
├── lib
│   ├── function
│   │   ├── Aes.go
│   │   ├── Rsa.go
│   │   ├── Rsa_test.go
│   │   └── Uuid.go
│   ├── httpLogger
│   │   └── HttpLogger.go
│   └── zlog
│       ├── SqlLog.go
│       └── ZapLogger.go
├── main.go
```
所以在测试文件的目录下肯定是找不到app.ini的，咋办呢？解决方法有很多

- copy一个配置到测试文件。这种方法最简单粗暴，但是太不灵活，测试用例可能在任何目录里面，这样搞有点难受

- 配置文件路径写成绝对路径。这种方法也不灵活，毕竟每个人的项目目录位置不一样，以后线上部署也麻烦

- 采用依赖注入的高级写法，测试的时候使用mock的方式注入配置。这种方法可以，也是比较好的方式，但是需要引入依赖注入组件，整个项目的架构需要更改，不推荐使用依赖注入把简单的问题复杂化。

- 跑测试的时候传入外部参数，依然不够灵活，而且麻烦

这个问题，我思考了很久，最终想了一个足够简单灵活的方式，代码如下：
```go
envFile := "app.ini"
// 读取配置文件, 解决跑测试的时候找不到配置文件的问题，最多往上找5层目录
for i := 0; i < 5; i++ {
    if _, err := os.Stat(envFile); err == nil {
        break
    } else {
        envFile = "../" + envFile
    }
}
conf, err := ini.Load(envFile)
if err != nil {
    log.Panicf("parse conf file [%s] failed, err: %s", envFile, err.Error())
}
```
使用一个for循环解决了这个问题，如果怕不够保险，可以改成10，大多数项目目录应该不会这么深，虽然不够优雅，但是还是相对比较简单的。

各位有什么更好的方法吗？有的话请留言指教
