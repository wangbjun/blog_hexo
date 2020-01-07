---
title: PHP设计模式之过滤器模式
date: 2017-01-02 09:00:00
category: 设计模式
tags: 
    - PHP
---

# 过滤器模式

> 过滤器模式（Filter Pattern）或标准模式（Criteria Pattern）是一种设计模式，这种模式允许开发人员使用不同的标准来过滤一组对象，通过逻辑运算以解耦的方式把它们连接起来。这种类型的设计模式属于结构型模式，它结合多个标准来获得单一标准。

日常生活中也有过滤器，这个比较容易理解，就是使用设备过滤出自己想要的，去掉那些不符合条件的。但是在编程里面是怎么实现的呢？

举个例子，有一组用户参与抽奖活动，我们需要筛选一部分符合条件的用户抽奖，其它不符合条件的用户咱直接提示未中奖！比如说需要注册时间大于3个月、消费金额大于100元、没有违规行为、活跃度大约500、性别为女。。。等条件！

有人说问为什么不使用数据库筛选，一条sql数据就搞定了啊，实际上有可能是因为这些数据并不是在一个表里面，有些数据可能需要计算得出。

<!--more-->

普通写法：

```php
循环所有用户...
if 注册时间 < 3个月 {
    出局;
}

if 消费金额 < 100 {
    出局;
}
```
这种写法没问题，写业务代码的时候大部分都是这么做，但是如果当你的业务逻辑十分复杂的时候，这样写容易乱，不容易维护。

下面展示使用过滤器模式的写法：

首先，我们需要一个用户类：

```php
<?php

namespace App\Filter;

class User
{
    private $id;

    private $name;

    private $age;

    private $gender;
    
    public function __construct($id, $name, $age, $gender)
    {
        $this->id     = $id;
        $this->name   = $name;
        $this->age    = $age;
        $this->gender = $gender;
    }

    /**
     * @return mixed
     */
    public function getName()
    {
        return $this->name;
    }

    /**
     * @param mixed $name
     */
    public function setName($name): void
    {
        $this->name = $name;
    }

    /**
     * @return mixed
     */
    public function getAge()
    {
        return $this->age;
    }

    /**
     * @param mixed $age
     */
    public function setAge($age): void
    {
        $this->age = $age;
    }

    /**
     * @return mixed
     */
    public function getGender()
    {
        return $this->gender;
    }

    /**
     * @param mixed $gender
     */
    public function setGender($gender): void
    {
        $this->gender = $gender;
    }
}
```
Filter接口：

```php
<?php

namespace App\Filter;

interface Filter
{
    public function filter(array $users): array;
}
```

AgeFiler:

```php
<?php

namespace App\Filter;

class AgeFilter implements Filter
{
    public function filter(array $users): array
    {
        $result = [];

        foreach ($users as $user) {
            if ($user->getAge() > 25) {
                $result[] = $user;
            }
        }

        return $result;
    }
}
```

MaleFilter:

```php
<?php

namespace App\Filter;

class MaleFilter implements Filter
{
    public function filter(array $users): array
    {
        $result = [];

        foreach ($users as $user) {
            if ($user->getGender() == '男') {
                $result[] = $user;
            }
        }

        return $result;
    }
}
```

最后使用：

```php
<?php
include "../include.php";

$users = [];

$users[] = new \App\Filter\User(1, "1", 25, "男");
$users[] = new \App\Filter\User(2, "2", 35, "男");
$users[] = new \App\Filter\User(3, "3", 27, "女");
$users[] = new \App\Filter\User(4, "4", 21, "男");
$users[] = new \App\Filter\User(5, "5", 24, "女");

$ageFilter = new \App\Filter\AgeFilter();
$result    = $ageFilter->filter($users);

$maleFilter = new \App\Filter\MaleFilter();
$result     = $maleFilter->filter($result);

var_dump($result);
```

这个例子比较简单，也不算太恰当，实际应用中，过滤器模式很多地方都用，比如PHP自带的就有一个Filter类,有一些类似 filter_input 这样的方法可以用来过滤变量。在laravel框架里面利用管道过滤器模式实现了middleware(中间件)，非常方便，在实现功能的同时增加了项目的可维护性。
