---
title: 二、你需要一个依赖注入容器吗？
date: 2017-02-04 13:10:08
category: 设计模式
tags: 
    - PHP
    - 依赖注入
---
>此文是本人翻译的来自国外某网站一篇文章 [Do you need a Dependency Injection Container?](http://fabien.potencier.org/do-you-need-a-dependency-injection-container.html)

>这篇文章是一系列关于依赖注入和PHP轻量级容器实现文章中的一部分：
[Part 1: What is Dependency Injection?](http://fabien.potencier.org/article/11/what-is-dependency-injection)
[Part 2: Do you need a Dependency Injection Container?](http://fabien.potencier.org/article/12/do-you-need-a-dependency-injection-container)
[Part 3: Introduction to the Symfony Service Container](http://fabien.potencier.org/article/13/introduction-to-the-symfony-service-container)
[Part 4: Symfony Service Container: Using a Builder to create Services](http://fabien.potencier.org/article/14/symfony-service-container-using-a-builder-to-create-services)
[Part 5: Symfony Service Container: Using XML or YAML to describe Services](http://fabien.potencier.org/article/15/symfony-service-container-using-xml-or-yaml-to-describe-services)
[Part 6: The Need for Speed](http://fabien.potencier.org/article/16/symfony-service-container-the-need-for-speed)

在这系列第一篇关于依赖注入的文章里，我尝试给出了一个依赖注入的web实例，今天我将要谈谈依赖注入容器。

首先，先看看一句大胆的言论：“大多数情况下，你不需要一个依赖注入容器也能受益于依赖注入” 

<!--more-->

但是当你有很多对象需要解决很多依赖，一个依赖注入容器就会非常有用（想一想那些框架）。如果你记得第一篇文章的例子，创建一个User对象需要先创建一个SessionStorage对象。这不是什么大问题，但是你在你创建所需对象之前至少需要知道你所需要的依赖：
```
$storage = new SessionStorage('SESSION_ID');
$user = new User($storage);
```
在接下来的文章里面，我将会谈谈Symfony2框架里面对依赖注入容器的实现。但是有一点我想说清楚，这种实现和Symfony框架本身没有什么关系，我会采用Zend框架里面例子来阐述。

Zend框架里面的Mail库简化了邮件操作，使用PHP mail() 函数就能发送邮件，但是这不太灵活。值得感谢的是，通过提供一个transport对象就可以很容易改变这点。下面的这段代码就展示了创建一个使用Gmail账号发送邮件的Zend_Mail对象：
```
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

一个依赖注入容器就是一个知道如何初始化和配置对象的容器。为了完成这个任务，它需要知道构造器参数和对象之间的关系。
下面是一段简单的硬编码的容器，针对之前的Zend_Mail例子：
```
class Container
{
  public function getMailTransport()
  {
    return new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
      'auth'     => 'login',
      'username' => 'foo',
      'password' => 'bar',
      'ssl'      => 'ssl',
      'port'     => 465,
    ));
  }

  public function getMailer()
  {
    $mailer = new Zend_Mail();
    $mailer->setDefaultTransport($this->getMailTransport());

    return $mailer;
  }
}
```
使用容器很简单：
```
$container = new Container();
$mailer = $container->getMailer();
```

当使用容器的时候，我们只需要一个mailer对象即可，我们不需要知道如何去创建，所有关于mailer对象创建的东西都被封装在容器内部。
但是，机智的读者可能会发现一个问题，容器里面都是硬编码的。所以我们需要进一步优化，添加一些参数，这样容器会更有用：
```
class Container
{
  protected $parameters = array();

  public function __construct(array $parameters = array())
  {
    $this->parameters = $parameters;
  }

  public function getMailTransport()
  {
    return new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
      'auth'     => 'login',
      'username' => $this->parameters['mailer.username'],
      'password' => $this->parameters['mailer.password'],
      'ssl'      => 'ssl',
      'port'     => 465,
    ));
  }

  public function getMailer()
  {
    $mailer = new Zend_Mail();
    $mailer->setDefaultTransport($this->getMailTransport());

    return $mailer;
  }
}
```
现在你就很容易通过构造函数传递用户名和密码等参数：
```
$container = new Container(array(
  'mailer.username' => 'foo',
  'mailer.password' => 'bar',
));
$mailer = $container->getMailer();
```
如果你需要改变mailer类来测试，对象的名字也可以通过参数来传递：
```
class Container
{
  // ...

  public function getMailer()
  {
    $class = $this->parameters['mailer.class'];

    $mailer = new $class();
    $mailer->setDefaultTransport($this->getMailTransport());

    return $mailer;
  }
}

$container = new Container(array(
  'mailer.username' => 'foo',
  'mailer.password' => 'bar',
  'mailer.class'    => 'Zend_Mail',
));
$mailer = $container->getMailer();
```
最后，但不是最少，每次我想要一个mailer，我不必新建一个实例，所以容器可以被改造成每次返回同一个对象：
```
class Container
{
  static protected $shared = array();

  // ...

  public function getMailer()
  {
    if (isset(self::$shared['mailer']))
    {
      return self::$shared['mailer'];
    }

    $class = $this->parameters['mailer.class'];

    $mailer = new $class();
    $mailer->setDefaultTransport($this->getMailTransport());

    return self::$shared['mailer'] = $mailer;
  }
}
```
通过引入一个静态的$shared属性，每次你调用getMailer()都会返回第一次创建的对象

容器包装了需要实现的基本特性，依赖注入容器管理对象：从初始化到配置。对象本身不知道自己被容器管理，更不知道容器。所以容器可以管理任何PHP对象，如果对象使用依赖注入解决依赖就更好了，但那不是先决条件。

当然，通过手动创建和维护容器类可能会成为噩梦。但是一个可以使用的容器的要求还是很低的，很容易实现。下一章我会介绍Symfony2框架里面依赖注入的实现
