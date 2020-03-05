---
title: 详解中间件设计模式
date: 2019-03-12 10:02:46
tags: 中间件
category: 编程开发
---

说到中间件（middleware），很多人应该都听说过，但是大体有2种意思，一种是一些衔接不同软件活系统的中间软件，比如说数据库中间件、消息中间件。另一种是在Web软件开发中代码层面的一种设计模式，比如说用户认证中间件、日志中间件，这些中间件的主要作用就是以一种集中统一、几乎无侵入的的方式去处理用户请求,而今天我们要讲的就是中间件设计模式。

<!--more-->

## 简介
说起中间件模式，估计很多人都想起来下面这张图，一个Web请求经过多个中间件的过滤，像pipeline一样处理这个请求，最终返回响应。

<img src = "/images/old/5f6e3e27ly1g3ow77jcovj20it0e9aas.jpg" />

中间件往往部署在路由的地方，用于统一过滤请求，举个例子，我们有一个特殊的服务，必须要求用户的年龄大于18岁，如果不使用中间件，我们传统的做法就是在每个请求的控制器或者方法里面做判断，从功能上说没啥问题，但是代码不够优雅，需要写很多重复代码，而且不利于维护，哪天我们要把这个年龄改成20岁呢？

<img src = "/images/old/5f6e3e27ly1g3owo9uj4hj20i30a1t9o.jpg" />

但是常见的23种设计模式里面并没有中间件模式，其实中间件是管道模式（也有人说是装饰模式）的一种实现，我也不知道为什么大部分框架都叫做中间件(middleware)...?

说个题外话，大部分设计模式主要就是为了解耦，提高代码可维护性和扩展性，并不是必须的，但是大部分情况下还是有益的。


## 管道模式
管道又称为pipeline，又叫流水线，工厂里面流水线大家应该都见过，一个产品需要经过很多道工序才能完成，比如苹果手机的一根数据线，大概有20多道工序，在工厂里面这些数据线会被放到传送带上面，依次完成各个工序，我们可以把一个请求看作是一个产品，流水线的每道工序看作是处理对象。

下面直接看代码：

1.Middleware.php
```php
<?php
interface Middleware
{
    public function execute(Closure $next);
}

```
2.LogMiddleware.php
```php
<?php
class LogMiddleware implements Middleware
{
    public function execute(Closure $next)
    {
        echo "Before Log!\n";
        $next();
        echo "After Log!\n";
    }
}

```
3.AuthMiddleware.php
```php
<?php
class AuthMiddleware implements Middleware
{
    public function execute(Closure $next)
    {
        echo "Before Check Auth!\n";
        $next();
        echo "After Check Auth!\n";
    }
}
```
4.Client.php
```php
<?php
class Client
{
    protected $middlewares = [];

    public function addMiddleware(Middleware $middleware)
    {
        $this->middlewares[] = $middleware;

        return $this;
    }

    public function getClosure()
    {
        return function ($current, $next) {
            return function () use ($current, $next) {
                return (new $next)->execute($current);
            };
        };
    }

    public function defaultHandler()
    {
        return function () {
            echo "开始处理!\n";
        };
    }

    public function handler()
    {
        call_user_func(array_reduce($this->middlewares, $this->getClosure(), $this->defaultHandler()));
    }
}
```
首先，我们定义了一个Middleware接口，规定了需要实现的方法，然后定义了多个具体实现类。有一个非常关键的地方就是这个方法的参数是有一个闭包函数，然后在实现类里面我们都必须调用这个方法。

最核心的代码在于Client类，首先它有一个成员变量，里面存储了多个实现了Middleware接口的对象，这个类里面最关键的方法就是getClosure，它返回一个闭包函数，这个闭包函数接受2个参数，这2个参数都是实现了Middleware接口的对象，但是这个闭包函数并没有立马执行。

其中一个非常关键的函数就是array_reduce,根据官方文档，array_reduce() 将回调函数 callback 迭代地作用到 array 数组中的每一个单元中，从而将数组简化为单一的值。先看一个非常简单的例子：
```php
$arr = [1, 2, 3, 4, 5];

$sum = array_reduce($arr, 'sum', 0);

function sum($a, $b)
{
  echo "before add: $a, $b\n";
  $sum =  $a + $b;
  echo "after add: $a, $b\n";
  return $sum;
}

var_dump($sum);
```
结果如下：
```php
before add: 0, 1
after add: 0, 1
before add: 1, 2
after add: 1, 2
before add: 3, 3
after add: 3, 3
before add: 6, 4
after add: 6, 4
before add: 10, 5
after add: 10, 5

int(15)
```
可见array_reduce会循环的把数组里面的数据两两代入函数，然后把返回的结果当作新的参数再次代入函数,最终会返回一个多层嵌套的闭包函数，然后通过call_user_func触发调用，这时候就会像拨洋葱一样，先从外面到里面，再从里面往外面。。。

上面的例子运行代码和结果如下：
```php
<?php
$client = new Client();

$client->addMiddleware(new LogMiddleware())
    ->addMiddleware(new AuthMiddleware());

$client->handler();
```
```php
Before Check age!
Before Log!
开始处理!
After Log!
After Check Age!
```
仔细看一下这个结果，是不是非常像第一张图那样，不过这个例子里面少了一个非常重要的request对象，这里纯粹只是展示中间件运行原理，完整的实战代码可以参考laravel框架里面的源码，实现原理差不多，只不过框架功能更加全面，考虑的东西更多。

