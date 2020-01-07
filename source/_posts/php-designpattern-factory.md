---
title: PHP设计模式之简单工厂模式
date: 2016-09-05 15:00:00
category: 设计模式
tags: 
    - PHP
---

> 这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。在工厂模式中，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。

---

工厂模式其实就是用来创建对象的，对象是什么的，当然是类的实例，在学习设计模式之前，请先回忆一下面向对象编程，牢记面向对象三大特性：封装，继承，多态！
很多语言并不是完全面向对象的，比如PHP，JS等脚本语言，但这并无妨碍我们去学习其设计思想，但是我们必须从面向对象的思路去理解这种设计！

在完全面向对象的世界里面，首先必须有类，然后才有对象，对象有属性和方法，在程序运行的时候我们需要先创建一个类，然后使用 **new** 去实例化一个类得到一个对象，然后去调用这个类的相关属性活方法。

<!--more-->

举个最简单的例子：
```php
<?php
class FileLog {
    
    public $file;
    
    public function log($param)
    {
        echo "Log $param to $this->file!\n";
    }
}

$logger = new FileLog();
$logger->file = "/data/log.log";
$logger->log("something");
```
大部分时候我们这么用就可以了，十分简单方便，但是使用建造中模式我们可以更灵活的创建对象，比如说我们系统里面支持2种日志记录方式，
一个是文件日志，一个是mongo日志，他们都有一个共同的功能那就是记录日志！可以采用如下设计：

我们先定义一个接口Log.php :
```php
<?php

namespace App\SimpleFactory;

interface Log
{
    public function log(string $param);
}

```
然后创建2个类实现这个接口：

FileLog.php
```php
<?php

namespace App\SimpleFactory;

class FileLog implements Log
{
    public function log(string $param)
    {
        echo "Log $param to File\n";
    }
}
```

MongoLog.php
```php
<?php

namespace App\SimpleFactory;

class MongoLog implements Log
{
    public function log(string $param)
    {
        echo "Log $param to Mongo\n";
    }
}
```
这时候我们需要一个工厂去生产这个日志对象 LogFactory.php：
```php
<?php

namespace App\SimpleFactory;

class LogFactory
{
    const FILE_LOG = 1;

    const MONGO_LOG = 2;

    public function getLogger(string $logType): Log
    {
        if ($logType == self::MONGO_LOG) {
            return new MongoLog();
        }

        return new FileLog();
    }
}
```
最后，新建一个demo.php验证是否可以使用，这里我使用了 **spl_autoload_register** 解决了文件加载问题:
```php
<?php

require "../include.php";

$factory = new \App\SimpleFactory\LogFactory();

$fileLog = $factory->getLogger($factory::FILE_LOG);
$fileLog->log("something");

$mongoLog = $factory->getLogger($factory::MONGO_LOG);
$mongoLog->log("something");


```
这种设计至少有一个好处可以灵活的切换日志类型，而且对于调用者来说他只需传一个参数即可，屏蔽了创建过程中的细节，实际应用中我们可以在创建对象的过程中做一些初始化工作！

接下来我们会讲一下工厂方法模式，这是简单工厂模式的改进版！
