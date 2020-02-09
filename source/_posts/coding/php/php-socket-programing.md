---
title: PHP Socket 网络编程
date: 2018-04-05 19:00:03
category: 编程开发
tags: 
    - PHP
    - Socket
---

## 前言
在做PHP开发的过程中，大部分我们都在和http协议打交道，在ISO模型里面，http属于应用层协议，它底层会用到TCP协议。http协议非常简单，它是一个文本协议，一个请求对应一个响应，客户端发起一个请求，服务端响应这个请求。http是一个一问一答的对话，每次请求都得重新建立对话（这里暂不讨论Keep-Alive），如果你想通过一个请求进行多次对话，那就是长连接通信，必须使用TCP或者UDP协议。

互联网运行的基石是建立在一些协议上的，目前而言主要是TCP/IP协议族，大部分协议都是公开开放的，计算机遵循这些协议我们才能通信，当然也有一些私有协议，私有协议只有自己知道如何去解析，相当来说更安全，比如QQ所用的协议就是自己定义的。在ISO模型里面，咱们常用的有http、ftp、ssh、dns等，但是不常用的数不胜数，发明一个协议不难，难的是如何设计的更好用，而且大家都喜欢用。

## Socket
Socket并不是一个协议，本质上说Socket是对 TCP/IP 协议的封装，它是一组接口，在设计模式中，Socket 其实就是一个门面（facade）模式，它把复杂的 TCP/IP 协议族隐藏在 Socket 接口后面，对用户来说，一组简单的接口就是全部，让 Socket 去组织数据，以符合指定的协议。

<!--more-->

下图展示了Socket在ISO模型里面大概位置：

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1g372hly7xkj20f20d8js4.jpg)

## PHP Socket
虽然PHP的强项是处理文本，一般用来写网页和http接口，但是官方依然提供了Socket扩展，编译PHP时在配置中添加--enable-sockets 配置项来启用，如果使用apt或yum安装，默认情况下是已启用。

[官方文档](https://www.php.net/manual/zh/book.sockets.php)里面列出了大概40个函数，但是常用的也就那几个，跟着文档，咱们一起来学学如何使用，首先声明一下，本人对Socket编程并不熟悉，如有错误的地方，希望大家指出来。

咱们先看一幅图，关于TCP客户端和服务端之间的通信过程，咱们平时写http接口的时候并未做这么多工作，那是客户端给封装好了：

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1g371bhib22j20da0dn0t2.jpg)


### 1.服务端代码
```php
<?php
set_time_limit(0);

$ip = '127.0.0.1';
$port = 8888;

$sock = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);

socket_bind($sock, $ip, $port);
socket_listen($sock, 4);

echo "Server Started, Listen On $ip:$port\n";

$accept = socket_accept($sock);

socket_write($accept, "Hello World!\n", 8192);

$buf = socket_read($accept, 8192);

echo "Receive Msg： " . $buf . "\n";

socket_close($sock);
```
简单说一下,为便于演示，所以省略了所有的错误处理代码，可以看到分为create、bind、listen、accept、write\read、close这几步，看上去非常简单！具体参数大家可以看一下文档！在服务端启动之后，当收到一个请求之后，我们首先返回了一个```Hello World\n```,然后又读取了8192个字节的数据，打印出来！最后关闭连接。

由于这里，咱还没有写客户端，所以暂时使用curl访问一下，运行效果如下：

===>服务端：

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1g373ivwa2ej20ls06274r.jpg)

===>客户端：

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1g373k4ahoij20hw03cwem.jpg)

从这个例子里面我们可以看出来，curl发出是一个标准的http请求，实际上它的每一行后面是有\n的，在http协议里面，这几行文本其实是头（header）,但是在这个例子里面，对于我们来说，它就是一段文本而已，服务端只是把它的内容打印出来了,并没有去按照http协议去解析。虽然我们返回了```Hello World！\n```，但是这也并没有按照http协议的格式去做，缺少响应头。我只能说curl比较强大，如果使用浏览器访问的话会失败，提示```127.0.0.1 sent an invalid response```。

但是稍加改造，我们就可以返回一个标准的http响应：
```php
$response = "HTTP/1.1 200 OK\r\n";
$response .= "Server: Socket-Http\r\n";
$response .= "Content-Type: text/html\r\n";
$response .= "Content-Length: 13\r\n\r\n";
$response .= "Hello World!\n";

socket_write($accept, $response, 8192);
```
这时候如果再用浏览器访问，就可以看到 Hello World!了，但是这个服务端目前是一次性的，就是说它只能处理一次请求，然后就结束了，正常的服务端是可以处理多次请求的，很简单，加一个死循环就行了！

只贴一下改动的部分，代码如下：
```php
while (true) {
    $accept = socket_accept($sock);

    $buf = socket_read($accept, 8192);

    echo "Receive Msg： " . $buf . "\n";

    $response = "HTTP/1.1 200 OK\r\n";
    $response .= "Server: Socket-Http\r\n";
    $response .= "Content-Type: text/html\r\n";
    $response .= "Content-Length: 13\r\n\r\n";
    $response .= "Hello World!\n";

    socket_write($accept, $response, 8192);

    socket_close($accept);
}
```
摇身一变，就是一个http服务了，使用ab测了一下，并发上万，是不是有点小激动？

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1g37ldkacn8j20l909twfu.jpg)


然而，之所以这么快是因为逻辑简单，假如你在while里面任何位置加一个 sleep(1) 你就会发现，原来这特么是串行的，一个个执行的，并不是并行，这段脚本一次只能处理一个请求！

解决这个问题方法有很多种，具体可以参考 [PHP并发IO编程之路](http://rango.swoole.com/archives/508), 看看前半段就行了，后半段是广告！该文章总结了3种方法：最早是采用多进程多线程方式，由于进程线程开销大，这种方式效率最低。后来演进出master-worker模型，也就是类似现在fpm采用的方式。目前最先进的方式就是异步io多路复用，基于epoll实现的。**理论上讲C能实现的，PHP都能通过扩展去实现**，而且PHP确实提供了相关扩展，其思想和C写的都差不多，然而今天咱不是说高并发编程的，还是接着说Socket吧！

### 2.客户端代码
之前的例子里面我们使用的是curl访问的，也可以使用浏览器或者telnet，这些工具都可以算作是客户端，客户端也可以自己实现。
```php
set_time_limit(0);

$port = 8888;
$ip = '127.0.0.1';

$sock = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);

echo "Connecting $ip:$port\n";

socket_connect($sock, $ip, $port);

$input = "Hello World Socket";

socket_write($sock, $input, strlen($input));

$out = socket_read($sock, 8192);

echo "Receive Msg: $out\n";

socket_close($sock);
```
这段代码同样省略了错误处理代码，可以看到第一步都是create，但是第二步变成connect，然后是read\write、最后close。

具体运行效果这里不再展示，和curl访问没多大区别，但是这个客户端也是一次性的，执行完了就结束！

## 实例
接下来，我们来写一个基于TCP通信的应用，这个应用非常简单，就是加减乘除！

(1)服务端代码：
```php
<?php
set_time_limit(0);

$ip = '127.0.0.1';
$port = 8888;

$sock = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);

socket_bind($sock, $ip, $port);

socket_listen($sock, 4);

echo "Server Started, Listen On $ip:$port\n";

while (true) {
    $accept = socket_accept($sock);

    $buf = socket_read($accept, 8192);

    echo "Receive Msg： " . $buf . "\n";

    $params = json_decode($buf, true);
    $m = $params['m'];
    $a = $params['a'];
    $b = $params['b'];

    switch ($m) {
        case '+';
            $response = $a + $b;
            break;
        case '-';
            $response = $a - $b;
            break;
        case '*';
            $response = $a * $b;
            break;
        case '/';
            $response = $a / $b;
            break;
        default:
            $response = $a + $b;
    }

    socket_write($accept, $response."\n", 8192);

    socket_close($accept);
}
```
(2)客户端代码：
```php
<?php
set_time_limit(0);

$port = 8888;
$ip = '127.0.0.1';

$sock = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);

echo "Connecting $ip:$port\n";

socket_connect($sock, $ip, $port);

$input = json_encode([
    'a' => 15,
    'b' => 10,
    'm' => '+'
]);

socket_write($sock, $input, strlen($input));

$out = socket_read($sock, 8192);

echo "Receive Msg: $out\n";

socket_close($sock);
```
在这些代码里面，我按照自己的需求定义了一个“协议”，我把需要运算的数和方式通过一个json数组传输，约定了一个格式，这个协议只有我自己清楚，所以只有我才知道怎么调用。服务端在接受到参数之后，通过运算得出结果，然后把结果返回给客户端。

但是这个例子还有问题，客户端依然是一次性的，参数都被硬编码在代码里面，不够灵活，最关键是没有用到TCP长连接的特性，我们每次计算都得重新发起请求、重新建立连接，实际上，我需要的是一次连接，多次对话，也就是进行多次计算！

目前为止，这些演示代码都没有复用连接，因为在服务端最后我close了这个连接，这意味着每次都是一个新的请求，如果是http服务的话尚且可以用一下，如何去实现一个TCP长连接呢？

## IO多路复用之select
select系统调用的目的是在一段指定时间内，监听用户感兴趣的文件描述符上的可读、可写和异常事件，虽然这个方式也比较低效，但是不妨了解一下，通过这种方式我们可以复用连接，完整的代码如下：
```php
<?php
set_time_limit(0);

$ip = '127.0.0.1';
$port = 8888;

$sock = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);

socket_bind($sock, $ip, $port);

socket_listen($sock, 4);

echo "Server Started, Listen On $ip:$port\n";

socket_set_nonblock($sock);

$clients = [];

while (true) {
    $rs = array_merge([$sock], $clients);
    $ws = [];
    $es = [];

    //监听文件描述符变动
    $ready = socket_select($rs, $ws, $es, 3);
    if (!$ready) {
        continue;
    }

    if (in_array($sock, $rs)) {
        $clients[] = socket_accept($sock);
        $key = array_search($sock, $rs);
        unset($rs[$key]);
    }

    foreach ($rs as $client) {
        $input = socket_read($client, 8096);
        if ($input == null) {
            $key = array_search($client, $clients);
            unset($clients[$key]);
            continue;
        }
        echo "input: " . $input;

        //解析参数，计算结果
        preg_match("/(\d+)(\W)(\d+)/", $input, $params);
        if (count($params) === 4) {
            $a = intval($params[1]);
            $b = intval($params[3]);
            $m = $params[2];
        } else {
            continue;
        }

        switch ($m) {
            case '+';
                $result = $a + $b;
                break;
            case '-';
                $result = $a - $b;
                break;
            case '*';
                $result = $a * $b;
                break;
            case '/';
                $result = $a / $b;
                break;
            default:
                $result = $a + $b;
        }

        $output = "output: $result\n";
        echo $output;
        socket_write($client, $output, strlen($output));
    }
}
```
然后我使用了telnet连接服务端进行操作，运行效果如下，一个基于TCP长连接的网络版简易计算器：

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1g385x7u95fj20p50f1wfh.jpg)

在这个例子，传参的“协议”稍微有点变化，只是为了更方便在telnet里面交互，但是很容易理解。这里面最关键是定义了一个全局变量用来存储连接资源描述符，然后通过select去监听变化,最后遍历整个数组，读取\写入数据！

## 总结
通过上面的简单介绍，希望大家都对PHP Socket编程有一些了解和认识，其实作为Web开发来说，很少会用到裸TCP去连接，大部分时候都是使用基于TCP的http协议，只有涉及到一些对响应速度要求非常高的应用，比如说游戏、实时通信、物联网才会用到，如果真的用到，不妨尝试一下Workman、Swoole这些成熟的框架！
