---
title: 一、什么是依赖注入？
date: 2017-02-01 19:10:00
category: 编程开发
tags: 
    - PHP
    - 依赖注入
---

>此文是本人翻译的来自国外某网站一篇文章 [What is Dependency Injection?](http://fabien.potencier.org/what-is-dependency-injection.html),第一次翻译，各位见谅

>这篇文章是一系列关于依赖注入和PHP轻量级容器实现文章中的一部分：
[Part 1: What is Dependency Injection?](http://fabien.potencier.org/article/11/what-is-dependency-injection)
[Part 2: Do you need a Dependency Injection Container?](http://fabien.potencier.org/article/12/do-you-need-a-dependency-injection-container)
[Part 3: Introduction to the Symfony Service Container](http://fabien.potencier.org/article/13/introduction-to-the-symfony-service-container)
[Part 4: Symfony Service Container: Using a Builder to create Services](http://fabien.potencier.org/article/14/symfony-service-container-using-a-builder-to-create-services)
[Part 5: Symfony Service Container: Using XML or YAML to describe Services](http://fabien.potencier.org/article/15/symfony-service-container-using-xml-or-yaml-to-describe-services)
[Part 6: The Need for Speed](http://fabien.potencier.org/article/16/symfony-service-container-the-need-for-speed)

今天，我一开始不会讲容器，我希望先通过一些具体的实例来介绍一下依赖注入的理念以及其所尝试解决的问题和它能给开发者带来的好处。如果你已经了解依赖注入，你可以跳过这篇文章去看下一篇。
依赖注入可能是我知道的最简单的设计模式之一，很可能你已经使用过，但是同时也是最难解释的，原因可能是大多数介绍依赖注入的文章用的例子都比较无聊。我想了一个比较适合PHP领域的例子，因为PHP主要用在web开发，所以让我们来看一个简单的web实例。
为了解决http协议无状态的问题，web应用需要一种在web请求之间记录用户信息的方法，简单的通过cookie或者session都能解决：
```
$_SESSION['language'] = 'fr';
```

<!--more-->

上面的代码把用户的语言存在了session变量里面。这样，对于同一个用户的请求，其所使用的语言就会被存储在$_SESSION数组里面，我们可以这样获取：
```
$user_language = $_SESSION['language'];
```
由于依赖注入只在面向对象的世界里有意义，我们假装我们有一个叫SessionStorage的类封装了处理session的方法：
```
class SessionStorage
{
  function __construct($cookieName = 'PHP_SESS_ID')
  {
    session_name($cookieName);
    session_start();
  }
  function set($key, $value)
  {
    $_SESSION[$key] = $value;
  }
  function get($key)
  {
    return $_SESSION[$key];
  }
  // ...
}
```
...和一个提供高级接口的易用的User类
```
class User
{
  protected $storage;

  function __construct()
  {
    $this->storage = new SessionStorage();
  }

  function setLanguage($language)
  {
    $this->storage->set('language', $language);
  }

  function getLanguage()
  {
    return $this->storage->get('language');
  }

  // ...
}
```
这些类足够简单，使用User类也非常容易：
```
$user = new User();
$user->setLanguage('fr');
$user_language = $user->getLanguage();
```
到目前为止，一切都很好...除非你想要更多的灵活性。万一你想要改变session里面的cookie名字呢？你可能会使用下面这些方法：
1. 在SessionStorage构造器里面硬编码名字
```
  class User
    {
      function __construct()
      {
        $this->storage = new SessionStorage('SESSION_ID');
      }

      // ...
    }
```
2. 在User类外面定义一个常量
```
 define('STORAGE_SESSION_NAME', 'SESSION_ID');
 class User
    {
      function __construct()
      {
        $this->storage = new SessionStorage(STORAGE_SESSION_NAME);
      }

      // ...
    }
```
3. 在User类构造器里面传递一个名字作为参数
```
 class User
    {
      function __construct($sessionName)
      {
        $this->storage = new SessionStorage($sessionName);
      }

      // ...
    }

    $user = new User('SESSION_ID');
```
4. 在User类构造器里面传递一个数组选项
```
    class User
    {
      function __construct($storageOptions)
      {
        $this->storage = new SessionStorage($storageOptions['session_name']);
      }

      // ...
    }

    $user = new User(array('session_name' => 'SESSION_ID'));
```

以上的所有选择都很烂，硬编码名字没有真正解决问题因为你以后可能随时会改变注意，你还得更改User类。使用常量也是一个坏注意，因为你又依赖了一个常量。通过传递一个数组参数可能是一个好的解决方案，但是依然不太好，它把User构造器和一个和它本身不相关的东西耦合了。
而且还有一个问题没法容易搞定：我怎么换掉SessionStorage类？比方说，用一个mock对象去测试，或者你想把session保存在数据库或内存里面。在目前的代码里面除非你更改User类，否则无法实现。

### 依赖注入
不要在User类里面创建SessionStorage对象，我们在类外面创建SessionStorage对象，然后通过构造函数把其作为参数传进来：
```
class User
{
  function __construct($storage)
  {
    $this->storage = $storage;
  }

  // ...
}
```
这就是依赖注入,就是这些！
```
$storage = new SessionStorage('SESSION_ID');
$user = new User($storage);
```
现在，配置一个session存储对象非常简单了，替换它也很容易，不用改变User类也可以实现其他功能。
 [Pico Container website](http://www.picocontainer.org/injection.html) 这样形容依赖注入：“依赖注入就是通过构造器、方法、属性获取所需要的元素”
依赖注入不仅仅局限于此：
- 构造器注入：
```
  class User
    {
      function __construct($storage)
      {
        $this->storage = $storage;
      }

      // ...
    }
```
- Setter注入：
```
  class User
    {
      function setSessionStorage($storage)
      {
        $this->storage = $storage;
      }

      // ...
    }
```
- 属性注入：
```
  class User
    {
      public $sessionStorage;
    }

    $user->sessionStorage = $storage;
```

一般来说，构造器注入最适合必要依赖，就像例子里面那样，Setter注入比较适合可选依赖，比如说缓存对象。当今，很多现代PHP框架大量使用依赖注入提供一系列既解耦又有凝聚力的组件：

```
// symfony: A constructor injection example
$dispatcher = new sfEventDispatcher();
$storage = new sfMySQLSessionStorage(array('database' => 'session', 'db_table' => 'session'));
$user = new sfUser($dispatcher, $storage, array('default_culture' => 'en'));
// Zend Framework: A setter injection example
$transport = new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
  'auth'     => 'login',
  'username' => 'foo',
  'password' => 'bar',
  'ssl'      => 'ssl',
  'port'     => 465,
));
$mailer = new Zend_Mail();
$mailer->setDefaultTransport($transport);
```

如果你想了解更多关于依赖注入的东西，我强烈建议你读一读[Martin Fowler introduction](http://www.martinfowler.com/articles/injection.html) 或者 [Jeff More presentation](http://www.procata.com/talks/phptek-may2007-dependency.pdf)。你也可以看看我去年关于依赖注入的[演讲](http://fabien.potencier.org/talk/19/decouple-your-code-for-reusability-ipc-2008),这里讲了更多细节

好了，就说这么多了，我希望你现在对依赖注入有更好的理解，本系列的下一章我会讲关于依赖注入容器
