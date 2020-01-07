---
title: PHP类的自动加载机制
date: 2016-01-05 12:00:00
category: PHP
tags: 
    - PHP
---

### 1.手动加载？
首先，咱们先想一个问题，为什么需要加载机制？

理论上讲，你可以把所有PHP代码都写在一个文件里面，早期PHP确实有很多这样的代码，因为那时候还没有面向对象的概念，没有代码分层，一个PHP文件里面一大堆函数，一个功能一个函数这么写就行了。

后来，大家都发现这样写起来太乱，不利于维护和扩展，更重要的是有了面向对象的概念，我们可以把属性和方法封装到类里面，然后需要用到的时候就实例化这个类，这从面向过程转向面向对象。

举个例子，现在在同一个文件夹里面有2个类文件 ClassA.php、ClassB.php，还有一个index.php:

<!--more-->

```php
<?php

$value = 100;

class ClassA
{
    public $name;
}
```

```php
<?php

class ClassB
{
    
}
```
```php
<?php
$a = new ClassA();

var_dump($a);
```
当你在index.php里面new ClassA 的时候没问题，至少这样写是没有语法错误的，现在IDE都很智能，可以自动提示。但是当你运行index.php的时候就会报错：
```php
PHP Fatal error:  Uncaught Error: Class 'ClassA' not found in /home/jwang/Documents/Work/MyBlog/PHP/Code/index.php:2
```
有些初学者就会很懵逼，明明这个类就在同一个文件夹下啊，为什么会找不到？

错误提示翻译过来就是: 在index.php 文件里面没有找到 ClassA！ 其实仔细想想，这个文件里面确实没有ClassA。

解决这个问题的方式就是使用 require 、include 关键字加载所需的类，其实PHP解释器在执行这个文件的时候遇到require或者include处理很简单，它就相当于下面这种写法：
```php
<?php

class ClassA
{
    public $name;
}

$a = new ClassA();

var_dump($a);
```
这时候当然不会报错，其实在MVC的框架里面，PHP执行的自始至终都是index.php这一个文件，只不过框架可以根据路由去加载不同的文件！所谓的加载，其实就是替换，如果没有这种加载机制，那如果你在不同的地方用到同一个类，是不是用到的地方就得copy一份类的代码...

有一点需要注意一下，require或include 不仅仅可以加载文件里面的类，也可以加载文件里面的变量，比如说 ClassA.php 这个文件里面还定义了一个```$value = 100 ```, 这时候你就可以在index.php里面直接使用$value了，它的值就是100；

假如说，这时候你在index.php里面也定义了一个$value, 而且值不一样，那就看你在哪里require的，假如你在require之前定义了$value，那就以require文件里面的为准，这就相当于重新赋值了。假如是之后，则以最新当前文件的定义为准！记住PHP代码是顺序执行的就行了

---
### 2.自动加载
虽然可以使用require，include去加载不同地方的类，但是还是太麻烦了，用到一个就得写一个，忘了就麻烦了，这时候就需要自动加载机制！

早期的时候可以使用 **__autoload()** 这个魔术方法去实现自动加载，但是现在一般都是建议使用 **spl_autoload_register**，当PHP脚本找不到所需的类时候就会自动调用这个函数！

比如说在上面index.php里面，由于我并没有定义这2个方法的任何一个，所以会报错, 修改一下：
```php
<?php

spl_autoload_register(function ($class) {
   require $class.".php";
});

$a = new ClassA();

var_dump($a);
```
可以定义多个spl_autoload_register 从不同的地方加载，而__autoload函数则不能重复定义。

需要注意的是，因为PHP是顺序执行的，这个函数必须在new之前定义，函数参数是一个函数，你也可以传一个匿名函数，只有一个参数，是需要加载的类的名称，由于这里并没有使用命名空间（后面再讲），所以在这里这个 $class 的值就是 ClassA。既然我们都知道类的名称了，我们就可以在这里去require或include所需的文件类。

>有一点需要说明一下，按标准来说，一个文件一个类，而且类的名字必须和文件名字一样（至少是有规律的），你要是不按这套路来，那可就没辙了！

---
### 3.命名空间
上面的例子里面，虽然解决了自动加载问题，但是依然有一个问题，假如现在多了一个文件夹App，我们把 ClassB 放到 App 文件夹里面，再运行代码，你会发现还是找不到ClassB，require会报错。

有好几种解决方法，你可以在require的时候判断文件是否存在，先在当前文件夹下找，如果不存在就去App文件找，把可能存在的地方都找一遍，如果都找不到就报错！

但是这种方法效率很低，有一些框架会采用文件命名带上文件夹名的方式解决这些问题，比如说把ClassB.php 改成 App_ClassB.php，这样就可以根据文件名找到文件，带来的后果就是文件夹名字超级长...不是太优雅！

默认情况下，如果你没有使用命名空间，所有文件都在同一个命名空间下全局(\), 使用命名空间来改造上面的代码：

ClassA.php:
```php
<?php
namespace App;

class ClassA
{
    public $name;
}
```
ClassB.php:
```php
<?php
namespace App;

class ClassB
{

}
```
这时候如果想在index.php里面使用ClassB，有两种选择，一种是写上命名空间```$a = new \App\ClassB()```,另一种则是使用user关键字:
```php
<?php
use App\ClassA;
use App\ClassB;

spl_autoload_register(function ($class) {
    require $class.".php";
});

$a = new ClassA();

$b = new ClassB();

var_dump($a);
```
但是这时候依然无法正常加载，因为这时候$class的值是 **App\ClassB**, 带上了命名空间，为了能正常加载我们需要改造一下自动加载函数：
```php
<?php
define("APP_PATH", __DIR__);

spl_autoload_register(function ($file) {
    $file = str_replace('\\', '/', trim($file, 'App'));
    include APP_PATH.'/app'.$file.".php";
});
```
简单的说，只是处理一下目录分隔符和路径！命名空间并不是为了解决自动加载问题，命名空间主要是为了解决类名重复问题，但是有命名空间我们就可以更容易的实现自动加载！

---
### 4.composer自动加载
现在很多框架更多的是使用composer实现自动加载，使用一些第三方的类库也更方便，即使我们不使用第三方类库，composer也很好用！基本步骤如下：

第一步，安装composer，这个我就不多说了，如果在Ubuntu下面，可以执行 ```sudo apt install composer```

第二步，在项目根目录执行 ```composer init``` 进行初始化操作，当然你可以自己新建一个composer.json 文件,如下：
```json
{
    "name": "root/auto-load",
    "authors": [
        {
            "name": "wangbenjun",
            "email": "wangbenjun@gmail.com"
        }
    ],
    "require": {}
}
```
第三步，配置自动加载选项，composer主要有一下2个配置选项用于配置自动加载, 其中dev是用于开发环境：
```json
 "autoload": {
        "classmap": [
            "lib"
        ],
        "psr-4": {
            "App\\": "app/"
        }
  },
 "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
  },
```
classmap 是类映射, 用于处理那些没有使用命名空间的类库文件夹，composer使用了一个关联数组给里面所有的文件做了一个映射，便于快速找到所需的类。psr-4 则是处理使用了命名空间的文件夹

修改完了之后执行 ```composer dumpautoload``` 就可以生成自动加载文件，可以看到多出一个vendor文件夹，在index.php里面require里面的autoload.php就可以实现自动加载了。

```php
<?php
require "vendor/autoload.php";

use App\ClassA;
use App\ClassB;

$a = new ClassA();

$b = new ClassB();

$c = new SomeClass();

var_dump($a);
var_dump($b);
var_dump($c);
```

最终目录结构如下图：

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1fx1uzv0zpuj208n0723yl.jpg)

现在大多数Web框架都是采用MVC模式，一般都有一个统一的入口文件，比如index.php, 通常会在这个入口文件定义自动加载机制，使用composer的话只需要include一个autoload.php文件即可！
