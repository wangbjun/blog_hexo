---
title: PHP设计模式之装饰器模式
date: 2016-11-21 11:20:08
category: 编程开发
tags: 
    - PHP
---

# 装饰器模式

>动态地给一个对象添加一些额外的职责，就增加功能来说，装饰器模式比生成子类更为灵活；它允许向一个现有的对象添加新的功能，同时又不改变其结构。

这个模式算是比较容易理解，它主要解决类的功能扩展问题，当我们需要给一个类增加功能时候，我们可以选直接修改当前类，也可以继承当前类然后增加新方法。但是装饰模式更强调的是动态扩展类的功能，而不是直接修改类的功能。

举个例子：大家早餐买煎饼果子的时候，可以选择加鸡蛋、辣条、火腿肠，但是也可以选择什么都不加。

<!--more-->

## 1.煎饼果子接口类：
```php
<?php

namespace App\Decorator;

interface IPancake
{
    public function price();

    public function cook();
}
```

## 2.具体的煎饼生产类：
```php
<?php

namespace App\Decorator;

class Pancake implements IPancake
{
    public function price()
    {
        return 5;
    }

    public function cook()
    {
        echo "制作煎饼...\n";
    }
}
```

## 3.煎饼装饰抽象类：
```php
<?php

namespace App\Decorator;

abstract class PancakeDecorator implements IPancake
{
    private $pancake;

    public function __construct(IPancake $pancake)
    {
        $this->pancake = $pancake;
    }

    public function cook()
    {
        $this->pancake->cook();
    }
}
```

## 4.具体装饰类一：
```php
<?php

namespace App\Decorator;

class EgeDecorator extends PancakeDecorator
{
    public function __construct(IPancake $pancake)
    {
        parent::__construct($pancake);
    }

    public function cook()
    {
        echo "加了一个鸡蛋...\n";
        parent::cook();
    }

    public function price()
    {
        return 1;
    }
}
```

## 5.具体装饰类二：
```php
<?php

namespace App\Decorator;

class HTCDecorator extends PancakeDecorator
{
    public function __construct(IPancake $pancake)
    {
        parent::__construct($pancake);
    }

    public function cook()
    {
        echo "加了一根火腿肠...\n";
        parent::cook();
    }

    public function price()
    {
        return 1.5;
    }
}
```

## 6.使用结果：
```php
<?php
require_once "../include.php";

$cake = new \App\Decorator\Pancake();
$cake->cook();

echo "-----------------------------------------\n";

$egeCake = new \App\Decorator\EgeDecorator($cake);
$egeCake->cook();

echo "-----------------------------------------\n";

$htcCake = new \App\Decorator\HTCDecorator($cake);
$htcCake->cook();

echo "-----------------------------------------\n";

$htcCake = new \App\Decorator\HTCDecorator($egeCake);
$htcCake->cook();

//结果如下：

制作煎饼...
-----------------------------------------
加了一个鸡蛋...
制作煎饼...
-----------------------------------------
加了一根火腿肠...
制作煎饼...
-----------------------------------------
加了一根火腿肠...
加了一个鸡蛋...
制作煎饼...
```

从这里看出，使用装饰模式，我们可以非常方便的动态扩展当前类的功能，可以随意组合装饰类实现类功能扩展！
