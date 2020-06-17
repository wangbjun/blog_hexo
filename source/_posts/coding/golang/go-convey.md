---
title: Golang第三方测试库GoConvey
date: 2020-04-13 21:00:33
tags: Golang
category: Golang
---

之前写过一篇Go测试的文章，文章介绍的是Go自带的官方测试库，今天来介绍一下一个非常流行的Go第三方测试库GoConvey，其实官方的文档已经写的非常清楚了，有兴趣的可以查看其 [Github主页](https://github.com/smartystreets/goconvey) ,里面有非常详细的介绍，英文好的话可以看看。

<!--more-->

## 1.快速开始
简而言之，GoConvey是一个完全兼容官方Go Test的测试框架，一般来说这种第三方库都比官方的功能要强大、更加易于使用、开发效率更高，闲话少说，先看一个example：
```go
package utils

import (
    . "github.com/smartystreets/goconvey/convey"
    "testing"
)

func TestSpec(t *testing.T) {
    Convey("Given some integer with a starting value", t, func() {
        x := 1
        Convey("When the integer is incremented", func() {
            x++
            Convey("The value should be greater by one", func() {
                So(x, ShouldEqual, 2)
            })
        })
    })
}
```
乍一看，这个写法有点奇怪，一层层的嵌套式，如果你使用IDE的话你可以点到源码里面看一下其方法注释，其实已经说的非常清楚了，这里摘取部分看一下：
```
// Convey is the method intended for use when declaring the scopes of
// a specification. Each scope has a description and a func() which may contain
// other calls to Convey(), Reset() or Should-style assertions. Convey calls can
// be nested as far as you see fit.
//
// IMPORTANT NOTE: The top-level Convey() within a Test method
// must conform to the following signature:
//
//     Convey(description string, t *testing.T, action func())
//
// All other calls should look like this (no need to pass in *testing.T):
//
//     Convey(description string, action func())
```
这个用法相对简单了，Convey定义了一个局部的作用域，在这个作用域里面我们可以定义变量，调用方法，然后重复继续这个操作，low-level的Convey会继承top-level的变量。

了解之后，我们来扩展一下这个例子：
```go
func TestSpec(t *testing.T) {
    Convey("Given some integer with a starting value", t, func() {
        x := 1
        y := 10
        Convey("When the integer is incremented", func() {
            x++
            Convey("The value should be greater by one", func() {
                So(x, ShouldEqual, 2)
            })
        })
        Convey("When x < y", func() {
            if x < y {
                x = x + y
                So(x, ShouldBeGreaterThan, y)
            }
        })
    })
}
```
非常简单，当然这里我们并没有测试任何函数或方法，下面咱们写一个函数真正测试一下，假设有下面的方法：
```go
func Div(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("can not div zero")
    }
    return a / b, nil
}
```
使用GoConvey的话，测试代码可以这么写：
```go
func TestDiv(t *testing.T) {
    const X = 10
    Convey("Normal Result", t, func() {
        res, err := Div(X, 2)
        So(res, ShouldEqual, 5)
        So(err, ShouldBeNil)
        Convey("Extend Scope", func() {
            res, err := Div(res, 2)
            So(res, ShouldEqual, 2)
            So(err, ShouldBeNil)
        })
    })
    Convey("Error Result", t, func() {
        res, err := Div(X, 0)
        So(res, ShouldEqual, 0)
        So(err, ShouldNotBeNil)
    })
}
```
有人可能会觉得这和官方的没多大区别，相当于多加了一个注释，可以对每一个测试用例标识，但是不仅仅如此，这个库还提供了大量增强的Assertions，可以非常方便的对字符串、slice、map结果进行断言测试，具体的话可以查看一下文档或者点进去看看源码注释，这些源码注释基本上已经写的非常清楚了。


## 2.Web UI
此外，框架还提供了一个Web端的UI界面，可以非常方便的查看测试覆盖和运行情况，还可以自动运行测试，执行```goconvey```命令就可以启动服务，快试一试吧！（虽然说像Goland这样的IDE也提供了GUI工具查看测试覆盖率，但是这个更加方便）

另外，这个框架还提供了自定义Assertions的功能，使用起来也很方便，有一个通用的模板：
```go
func should<do-something>(actual interface{}, expected ...interface{}) string {
    if <some-important-condition-is-met(actual, expected)> {
        return ""   // empty string means the assertion passed
    }
    return "<some descriptive message detailing why the assertion failed...>"
}
```
举个例子，这里定义一个试试：
```go
func shouldNotGreatThan100(actual interface{}, expected ...interface{}) string {
    if actual.(int) > 100 {
        return "too big than 100"
    } else {
        return ""
    }
}
```
具体使用起来和库里面其它的Assertions是完全一模一样的，没有任何区别，这里就不演示了。

## 3.定义通用的逻辑
有时候测试会需要做一些准备工作，而且是重复的，比如说一些初始化操作，这时候就可以定义一个函数完成这件事，不必每次测试重复做，官方文档里面举了一个数据库测试的例子，每次测试前开启事务，测试结束后回滚事务，这里贴一下官方的example，大家看一下，很容易理解：
```go
package main

import (
    "database/sql"
    "testing"

    _ "github.com/lib/pq"
    . "github.com/smartystreets/goconvey/convey"
)

func WithTransaction(db *sql.DB, f func(tx *sql.Tx)) func() {
    return func() {
        tx, err := db.Begin()
        So(err, ShouldBeNil)

        Reset(func() {
            /* Verify that the transaction is alive by executing a command */
            _, err := tx.Exec("SELECT 1")
            So(err, ShouldBeNil)

            tx.Rollback()
        })

        /* Here we invoke the actual test-closure and provide the transaction */
        f(tx)
    }
}

func TestUsers(t *testing.T) {
    db, err := sql.Open("postgres", "postgres://localhost?sslmode=disable")
    if err != nil {
        panic(err)
    }

    Convey("Given a user in the database", t, WithTransaction(db, func(tx *sql.Tx) {
        _, err := tx.Exec(`INSERT INTO "Users" ("id", "name") VALUES (1, 'Test User')`)
        So(err, ShouldBeNil)

        Convey("Attempting to retrieve the user should return the user", func() {
             var name string

             data := tx.QueryRow(`SELECT "name" FROM "Users" WHERE "id" = 1`)
             err = data.Scan(&name)

             So(err, ShouldBeNil)
             So(name, ShouldEqual, "Test User")
        })
    }))
}

/* Required table to run the test:
CREATE TABLE "public"."Users" ( 
    "id" INTEGER NOT NULL UNIQUE, 
    "name" CHARACTER VARYING( 2044 ) NOT NULL
);
*/
```

基本上就是这些，强烈建议大家看一下官方文档，有很多细节这里并没有提到，毕竟咱也不能照抄，这个库完全可以代替官方的库，简单易用，效率高，推荐使用。
