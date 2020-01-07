---
title: PHP设计模式之单例模式
date: 2016-11-25 18:00:00
category: 设计模式
tags: 
    - PHP
---

>>单例模式（Singleton Pattern）是最简单的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
 这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。
----

最大好处是减少了内存的开销，尤其是频繁的创建和销毁实例，而且可以避免对一些资源的多重占用，对于PHP Web应用来说，虽然每次请求结束之后所有对象都会被销毁，但是依然有意义。

举个例子，一个请求有好几个操作，必须调用好几个对象的不同方法，刚好在这个几个方法里面都会用到redis，如果不使用单例模式，那么在每个对象里面都会重新实例化一次redis，浪费内存和时间，建立网络连接也耗时，但是如果使用单例，则只需要在第一次调用redis的使用实例化对象。

<!--more-->

这里就拿Redis举例，私有化构造方法和clone方法，然后提供一个静态方法用于实例化对象：
```php
<?php

namespace App\Single;

class Redis
{
    private function __construct()
    {
        echo "Connect to redis...\n";
    }

    private function __clone()
    {
    }

    private static $instance = null;

    public static function getRedis()
    {
        if (static::$instance == null) {
            static::$instance = new static();
        }

        return static::$instance;
    }

    public function showMsg()
    {
        echo "Hello World\n";
    }
}
```
调用的时候：
```php
<?php

require "../include.php";

$redis = \App\Single\Redis::getRedis();

$redis->showMsg();

$redis = \App\Single\Redis::getRedis();

$redis->showMsg();

$redis = \App\Single\Redis::getRedis();

$redis = \App\Single\Redis::getRedis();

$redis->showMsg();
```

结果：
```php
Connect to redis...
Hello World
Hello World
```

从结果可以看到，无论你调用几次，构造方法只会只会执行一次, 说明redis类只实例化了一次。在Java里面可能还有饿汉式和懒汉式之分，
不过在PHP里面由于语言特性限制，实际上只有一种模式，就是上面的饿汉式，而且PHP也不存在多线程问题（原生不支持多线程）！

有个小坑，需要说一下，PHP有多进程的扩展pcntl 和 多线程扩展pthread，如果你使用了这些扩展的话，最好不要使用单例模式，这会带来争用问题，
产生很多意想不到的结果，而PHP又没提供类似Java那样的synchronized关键字用于加锁，所以咯，最简单的方法就是在每个进程\线程里面单独实例化。
