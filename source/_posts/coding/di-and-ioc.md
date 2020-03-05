---
title: PHP依赖注入和控制反转
date: 2018-07-13 22:15:05
tags: PHP
category: 编程开发
---

这2个其实都算得上是一种设计模式或者说是一种软件设计思想，目的都是为了增加软件可维护性和扩展性，比如在Java Web框架SpringMVC 和PHP Web框架laravel里面都有应用。

首先得理解什么叫依赖？从宏观上看，得益于开源软件运行的兴起，很多时候我们写项目并不是什么都是从零开始，我们往往会利用很多现成的开源代码进行快速开发，能不重复造轮子最好，所以我们往往依赖很多开源组件。gradle、npm、composer 等工具的部分功能就是解决项目依赖问题。

<!--more-->

从微观上看，在实际写代码里面，对象与对象之间也会产生依赖关系，比如一个数据库查询类需要用到一个数据库连接、一个文章评论类用到一个文章，这里的依赖主要指对象之间的关系。

举个栗子，在一个 SessionService 里面你需要一个 FileSession ：

## 普通写法：

```php
class FileSession
{
    private $file;

    ... more code

    public function set($name, $value)
    {
        echo "set $name = $value into $this->file\n";
    }

    public function get($name)
    {
        echo "get $name value\n";
    }
}
```

service类：

```php
class SessionService
{
    private $sessionHandler;

    public function __construct()
    {
        $this->sessionHandler = new FileSession()；
    }

    public function set($name, $value)
    {
        $this->sessionHandler->set($name, $value);
    }

    public function get($name)
    {
        return $this->sessionHandler->get($name);
    }

    ...more code
}
```

在这种普通写法里面，当我们需要一个 sessionHandler 的时候我们是直接在构造函数里面实例化，这样没啥问题，确实解决了依赖问题。但是依赖注入的另一个词“注入”更强调的是一种从外部而来的，而不是内部。

<img src = "/images/old/5f6e3e27ly1fxq63pd6oej20ct0c03ze.jpg" />

改造如下：

## 依赖注入写法：

```php
class SessionService
{
    private $sessionHandler;

    public function __construct($sessionHandler)
    {
        $this->sessionHandler = $sessionHandler；
    }

    public function set($name, $value)
    {
        $this->sessionHandler->set($name, $value);
    }

    public function get($name)
    {
        return $this->sessionHandler->get($name);
    }

    ...more code
}
```

这种写法要求你在使用service的时候从外部传入一个handler，这就实现了依赖注入，注入的方式有很多种，刚才这种可以称之为构造器注入，还有一种叫setter注入，比如说，我们可以在service里面里面提供一个setter函数用于设置所需的handler：

```php
public function setSessionHandler($sessionHandler)
{
    $this->sessionHandler = $sessionHandler
}
```

这种写法有哪些好处呢？一个是解耦，假如说这个FileSession实例化的时候还需要其它操作，比如传入一个配置参数，原本的写法可能就需要更改service类了，在构造函数里面啪啪啪写一堆。还有就是方便测试，既然解耦了就可以很方便的进行单元测试。另一个是控制反转，就是说这个FileSession外部传入的，是service类无法控制的，也就说控制权在于外部。

很多软件在设计的时候都采用分层结构，最典型的就是计算机网络，Http协议依赖TCP协议，层与层之间通过约定的的接口进行交互，既减少了代码的复杂度，也提高了可维修性。比如说你哪一天重构了FileSession，没问题，只要你保证所有方法的返回结果和之前一样就行。

为了更灵活的运用这种注入机制我们可能需要采用一个接口去约束，举个例子，我们先增加一个接口sessionHandler：

```php
interface SessionHandler
{
    public function set($name, $value);

    public function get($name);
}
```

我们约定，只要你实现了这个接口，你就可以当一个sessionHandler，你就可以用来处理session，至于你怎么实现，service不管，比如说我们换一个redis：

```php
class RedisHandler implments SessionHandler
{
    private $redisInstance;

    public function __construct()
    {
        $this->redisInstance = new Redis();
    }
    public function set($name, $value)
    {
        $this->redisInstance->set($name, $value);
    }

    public function get($name)
    {
        return $this->redisInstance->get($name);
    }
}
```

这时候我们可以在service的构造函数稍作修改，增加一个类型约束：

```php
 public function __construct(SessionHandler $sessionHandler)
 {
        $this->sessionHandler = $sessionHandler；
 }
```

这样的设计之后，好处显而易见，我们可以很轻松替换掉之前的fileSession，不改动service的一行代码，只要按照sessionHandler的接口去实现相应的方法就行，在laravel里面这样的接口就叫做 Contracts，下面就是框架里面的Cache缓存的 Contracts：

```php
<?php

namespace Illuminate\Contracts\Cache;

interface Store
{
    /**
     * Retrieve an item from the cache by key.
     *
     * @param  string|array  $key
     * @return mixed
     */
    public function get($key);

    /**
     * Retrieve multiple items from the cache by key.
     *
     * Items not found in the cache will have a null value.
     *
     * @param  array  $keys
     * @return array
     */
    public function many(array $keys);

    /**
     * Store an item in the cache for a given number of minutes.
     *
     * @param  string  $key
     * @param  mixed   $value
     * @param  float|int  $minutes
     * @return void
     */
    public function put($key, $value, $minutes);

    /**
     * Store multiple items in the cache for a given number of minutes.
     *
     * @param  array  $values
     * @param  float|int  $minutes
     * @return void
     */
    public function putMany(array $values, $minutes);

    ... more code
}
```

据我看到的，在laravel框架里面自带了至少5种实现，分别是Array、File、Database、Memcached、Redis, 如果你愿意你也可以自己去实现这个 Contracts，然后替换到框架里面的，不过框架本身实现的已经非常优秀了，除非你写的更好，一般情况下不需要这样做，但是laravel提供了这种可能。
同样，在laravel框架里面session自带了Cache，Database，File这种几种实现，可以随意切换。

<img src = "/images/old/5f6e3e27ly1fxq0xh7s02j20as0hiaaq.jpg" />

---

## IOC容器

说了最后，必须再说说IOC容器，IOC核心思想是通过IoC容器管理对象的生成、资源获取、销毁等生命周期，在IoC容器中建立对象与对象之间的依赖关系，IoC容器启动后，所有对象直接取用，调用层不再使用new操作符产生对象和建立对象之间的依赖关系。

简单理解就是不再使用new创建对象了，而且使用容器来管理对象，需要对象就从容器里面取，而且你只需要在参数上声明依赖，容器就直接给你对象了，炒鸡方便，比如在laravel里面，有很多这样的写法：

```php
public function comment(Post $post, Request $request)
{
    $this->validate($request, [
        'content' => 'required|min:5'
    ]);

    $comment = new Comment([
        'content' => $request->get('content'),
        'user_id' => auth()->user()->id,
        'post_id' => $post->id,
    ]);

    $post->comments()->save($comment);

    return redirect()->back();
}
```

我们只需要在方法的参数上面标明所需的方法，就可以在代码直接用了，ioc容器替我们自动注入了依赖！

