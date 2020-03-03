---
title: 三、Symfony服务容器介绍
date: 2017-02-05 18:10:00
category: 编程开发
tags: 
    - PHP
    - 依赖注入
---

>此文是本人翻译的来自国外某网站一篇文章 [Do you need a Dependency Injection Container?](https://fabien.potencier.org/do-you-need-a-dependency-injection-container.html)

>这篇文章是一系列关于依赖注入和PHP轻量级容器实现文章中的一部分：
[Part 1: What is Dependency Injection?](https://fabien.potencier.org/article/11/what-is-dependency-injection)
[Part 2: Do you need a Dependency Injection Container?](https://fabien.potencier.org/article/12/do-you-need-a-dependency-injection-container)
[Part 3: Introduction to the Symfony Service Container](https://fabien.potencier.org/article/13/introduction-to-the-symfony-service-container)
[Part 4: Symfony Service Container: Using a Builder to create Services](https://fabien.potencier.org/article/14/symfony-service-container-using-a-builder-to-create-services)
[Part 5: Symfony Service Container: Using XML or YAML to describe Services](https://fabien.potencier.org/article/15/symfony-service-container-using-xml-or-yaml-to-describe-services)
[Part 6: The Need for Speed](https://fabien.potencier.org/article/16/symfony-service-container-the-need-for-speed)

在依赖注入这系列文章里，之前我们已经谈过一些基本思想。前面2篇文章介绍的东西对于更好的理解我们接下来文章要说的非常重要，现在是时候了解Symfony2里面服务容器的实现了。
Symfony里面依赖注入容器是被一个叫sfServiceContainer的类管理的，这是一个非常轻的类，它实现了我们上篇文章里面说到的基本特性。
在Symfony里面，一个服务就是一个被容器管理的对象。在上一篇文章Zend_Mail例子里，我们有2个：mailer服务和mail_transport服务：

<!--more-->

```
class Container
{
  static protected $shared = array();

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

如果我们让Container类继承sfServiceContainer类，就可以简化一下代码：
```
class Container extends sfServiceContainer
{
  static protected $shared = array();

  protected function getMailTransportService()
  {
    return new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
      'auth'     => 'login',
      'username' => $this['mailer.username'],
      'password' => $this['mailer.password'],
      'ssl'      => 'ssl',
      'port'     => 465,
    ));
  }

  protected function getMailerService()
  {
    if (isset(self::$shared['mailer']))
    {
      return self::$shared['mailer'];
    }

    $class = $this['mailer.class'];

    $mailer = new $class();
    $mailer->setDefaultTransport($this->getMailTransportService());

    return self::$shared['mailer'] = $mailer;
  }
}
```
这还不够，但是这会给我们一个稍微强大和干净的接口。我们做了以下改变：
- 所有的方法名都以Service为后缀.为了方便，一个服务名字必须以get开始，Service结束，每一个服务都有一个唯一的标识符。通过定义一个getMailTransportService()方法，我们就相当于定义了一个名叫mail_transport的服务。
- 所有方法都是protected.我们待会就会知道如何从容器取得一个服务了
- 容器可以通过数组来获取参数值 ($this['mailer.class'])

让我们看看如何使用新的容器类：
```
require_once 'PATH/TO/sf/lib/sfServiceContainerAutoloader.php';
sfServiceContainerAutoloader::register();

$sc = new Container(array(
  'mailer.username' => 'foo',
  'mailer.password' => 'bar',
  'mailer.class'    => 'Zend_Mail',
));

$mailer = $sc->mailer;
```
现在，因为Container类已经继承了sfServiceContainer类，接口就清晰多了：
- 服务可以通过统一的接口获取：
```
    if ($sc->hasService('mailer'))
    {
      $mailer = $sc->getService('mailer');
    }

    $sc->setService('mailer', $mailer);
```
- 作为捷径，服务也可以通过属性符号获取
```
   if (isset($sc->mailer))
    {
      $mailer = $sc->mailer;
    }

    $sc->mailer = $mailer;
```
- 参数也可以通过统一的接口获取
```
 if (!$sc->hasParameter('mailer_class'))
    {
      $sc->setParameter('mailer_class', 'Zend_Mail');
    }

    echo $sc->getParameter('mailer_class');

    // Override all parameters of the container
    $sc->setParameters($parameters);

    // Adds parameters
    $sc->addParameters($parameters);
```
-  作为捷径，参数也可以通过数组获取
```
 if (!isset($sc['mailer.class']))
    {
      $sc['mailer.class'] = 'Zend_Mail';
    }

    $mailerClass = $sc['mailer.class'];
```
- 你还可以迭代获取容器里面的所有服务
```
  foreach ($sc as $id => $service)
    {
      echo sprintf("Service %s is an instance of %s.\n", $id, get_class($service));
    }
```

当你有少量服务需要管理的话使用sfServiceContainer类非常有用方便，即使这样你还得做很多基础工作。如果你需要管理的服务增长到一定数量级，我们就需要一种更好的方式。

这也就是为什么大多数时候你并不需要直接使用sfServiceContainer类，但是它作为Symfony依赖注入容器实现的基石，我们还是需要花一些时间去描述它。

在下一篇文章里面，我们会看一下sfServiceContainerBuilder类，它简化了服务定义的过程。
