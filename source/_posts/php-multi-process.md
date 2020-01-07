---
title: PHP多进程编程应用
date: 2017-06-09 15:10:03
category: 多进程
tags: 
    - PHP
---

在日常开发中，我们经常会遇到需要使用脚本处理一些数据，在数据量比较大的情况下，我们可以采用并行的方式处理，比如说：

## 1.启动多个实例

这种方式简单实用，推荐，比如说使用下面的shell脚本我们就可以轻松的启动多个进程去处理

```
#!/bin/bash
  
for((i=1;i<=8;i++))
do   
    /usr/bin/php multiprocessTest.php &
done
  
wait
```
但是这种方式依赖外部工具，不够灵活！其实我们也可以采用多进程|多线程的方式

<!--more-->

PHP提供了大量关于进程相关的扩展，大部分都是和linux系统编程相关，我觉得应该就是对C库api的调用，文档也基本上没写，如果想用好估计得对linux系统下C编程非常熟悉！

其中pthreads是多线程需要用到的，多进程会用到pcntl和posix扩展，这篇文章就是简单介绍一下这两个扩展的应用。


## 2.启用多进程
php多进程需要pcntl和posix扩展支持，可以通过 php -m 查看是否安装，需要注意的是目前多进程实现只能在cli模式下使用，虽然是个残废，不妨也了解一下，具体的API可以查看官方文档，这里先举个简单的例子：
```
<?php
foreach (range(1, 5) as $index) {
    $pid = pcntl_fork();
    if ($pid === -1) {
        echo "failed to fork!\n";
        exit;
    } elseif ($pid) {
        $pid = posix_getpid();
        pcntl_wait($status); //父进程必须等待一个子进程退出后，再创建下一个子进程。
        echo "I am the parent, pid: $pid\n";
    } else {
        $cid = posix_getpid();
        echo "fork the {$index}th child, pid: $cid\n";
        exit; //必须
    }
}
```
这个例子很简单，循环了5次，在每次循环的时候创建一个进程，然后打印一句话

主要使用的方法就是函数 pcntl_fork()，一次调用两次返回，在父进程中返回子进程pid，在子进程中返回0，出错返回-1

posix_getpid()函数是返回当前进程 id，pcntl_wait()是等待或返回fork的子进程状态，pcntl_wait()将会存储状态信息到status 参数上，这个通过status参数返回的状态信息可以通过其它函数获得。

其中执行结果如下，在不同的机器是pid不一样：

```
I am the parent, pid: 11183
fork the 1th child, pid: 11184
I am the parent, pid: 11183
fork the 2th child, pid: 11185
I am the parent, pid: 11183
I am the parent, pid: 11183
I am the parent, pid: 11183
fork the 3th child, pid: 11186
fork the 5th child, pid: 11188
fork the 4th child, pid: 11187
```

**第一个注意点**:

如果是在循环中创建子进程,那么子进程中最后要exit,防止子进程进入循环!

**第二个注意点**:

这个和go的协程有点类型，主进程必须等待子进程执行完任务, 如果你不等待，你会发现一个是执行的顺序不固定，第二个打印的记录会少于10条，原因很简单，子进程还没来得及打印就结束了。

有一个简单方法是使用 pcntl_wait()，但是你会发现上面这个例子完全变成并行了...上面的结果就是，无论你运行多少次，每次都是按照1到5的顺序打印，这和我们多进程的所要实现的效果有点差异，我们需要的应该是1和5并行！

下面这种写法就可以实现这种效果：
```
<?php
 
$ids = [];
 
foreach (range(1, 5) as $index) {
    $ids[] = $pid = pcntl_fork();
    if ($pid === -1) {
        echo "failed to fork!\n";
        exit;
    } elseif ($pid) {
        $pid = posix_getpid();
        echo "I am the parent, pid: $pid\n";
    } else {
        $cid = posix_getpid();
        echo "fork the {$index}th child, pid: $cid\n";
        exit;
    }
}
 
foreach ($ids as $i => $pid) {
    if ($pid) {
        pcntl_waitpid($pid, $status);
    }
}
 
 
结果如下：
fork the 1th child, pid: 8392
I am the parent, pid: 8390
I am the parent, pid: 8390
fork the 2th child, pid: 8393
I am the parent, pid: 8390
I am the parent, pid: 8390
I am the parent, pid: 8390
fork the 3th child, pid: 8394
fork the 4th child, pid: 8395
fork the 5th child, pid: 8396
```

多次运行你会发现，每次的打印顺序都不一样，这就说明了1到5是并行执行的，也就是实现了多进程的效果！

其中pcntl_waitpid() 作用是等待或返回fork的子进程状态，挂起当前进程的执行直到参数pid指定的进程号的进程退出， 或接收到一个信号要求中断当前进程或调用一个信号处理函数

在这段代码里面，我们提前准备了一个数组存放这些子进程的pid，然后使用一个循环不停的查询其状态等待其结束！倘若你在上面的代码里面在子进程里面加一个随机的sleep，如下：
```
$cid = posix_getpid();
$t = random_int(1,20);
sleep($t);
echo "fork the {$index}th child, pid: $cid, wait: $t\n";
exit;
然后运行结果如下：

I am the parent, pid: 8772
I am the parent, pid: 8772
I am the parent, pid: 8772
I am the parent, pid: 8772
I am the parent, pid: 8772
fork the 1th child, pid: 8773, wait: 1
fork the 4th child, pid: 8776, wait: 5
fork the 3th child, pid: 8775, wait: 14
fork the 2th child, pid: 8774, wait: 16
fork the 5th child, pid: 8777, wait: 18
```

## 3.父进程和子进程之间关系？

子进程是复制了父进程的代码和内存空间，这意味着如果你在父进程里面定义了一些变量，在子进程里面也是可以操作访问的，这同时也意味着如果多个子进程操作同一个变量必然会出现覆盖和争用问题

比如说同时修改一个变量、同时往一个文件写入内容，需要通过锁机制保证同一时刻只能有一个进程操作。

还有一些坑，假如你在父进程去实例化一个mysql连接，在多个子进程里面同时使用，也会出现争用问题，所以涉及到这类资源类的变量，务必在各个子进程内部单独创建！

## 4.进程信号
进程信号也是linux操作系统的一些概念，这里就说说在PHP里面关于信号的一个应用

有些项目里面有时候会用到一些脚本，比如处理redis队列的脚本，通常的做法是写一个while循环从队列里面不停的取出数据处理，为了防止内存泄露或者进程假死，一般都会定时的重启脚本，通过做法就是先终止脚本再启动脚本，但是做的不好可能会导致数据丢失

举个例子，假如你这个脚本刚好从redis取出一条数据，然后正在处理中，操作还未完成，你突然终止脚本，那这个数据就丢失了。

使用信号注册我们可以更加优雅的重启或者终止脚本，你可以称之为平滑重启！看一下下面的代码:
```
<?php
 
//ctrl+c
pcntl_signal(SIGINT, function () {
    fwrite(STDOUT, "receive signal: " . SIGINT . " do nothing ...\n");
});
 
//kill
pcntl_signal(SIGTERM, function () {
    fwrite(STDOUT, "receive signal: " . SIGTERM . " I will exit!\n");
    exit;
});
 
while (true) {
    pcntl_signal_dispatch();
    echo "do something。。。\n";
    sleep(5);
}
```
Linux进程信号分为很多种，PHP里面定义了43种，咱就说说常用的几种：

SIGINT 2 这个其实相对于 ctrl+c

SIGTERM 15 就是 kill 默认的参数，表示终止进程

SIGKILL 9 就是 kill -9, 表示立马终止，这个信号在PHP里面是无法注册的

 

所谓注册信号就是接管系统对这个信号的处理方式，如果你不注册这个信号，进程就会按照默认方式去处理这个信号，如果你在代码里面注册这个信号，你就可以自定义处理方式，比如说在脚本里面先处理完当前数据，然后再退出！

看明白了这个就可以读懂上面的例子了，其中 pcntl_signal 是注册信号处理handler，第一个参数是你需要注册的信号，第二个是处理操作，可以是匿名函数或者一个函数名，可以注册多个信号

pcntl_signal_dispatch 调用每个等待信号通过pcntl_signal() 安装的处理器。早期PHP还有一种写法是使用 ticks，性能非常差，php5.3之后建议都使用 pcntl_signal_dispatch。

说明一下：pcntl_signal()函数仅仅是注册信号和它的处理方法，真正接收到信号并调用其处理方法的是pcntl_signal_dispatch()函数必须在循环里调用，为了检测是否有新的信号等待dispatching。



## 5.应用场景

由于进程的系统开销还是比较大，一般不太适合拿来做大规模并发程序，使用线程或者协程可能更好，拿来写个3-5个进程的后台脚本倒是有点用！比如说写个爬虫同时爬取多个网站的数据！举个例子：
```
<?php
$urls = [
    'https://www.baidu.com',
    "https://www.mi.com",
    "https://www.qingyidai.com"
];
 
$ids = [];
 
foreach ($urls as $url) {
    $ids[] = $pid = pcntl_fork();
    if ($pid === -1) {
        echo "failed to fork!\n";
        exit;
    } elseif ($pid) {
    } else {
        echo "start get url: ".$url."\n";
        crawler($url);
        exit;
    }
}
 
//爬取网页，取出网页标题
function crawler($url)
{
    $content = file_get_contents($url);
 
    preg_match("/<title>(.*)<\/title>/", $content, $matches);
 
    echo $matches[1]."\n";
}
 
foreach ($ids as $i => $pid) {
    if ($pid) {
        pcntl_waitpid($pid, $status);
    }
}
  
运行结果如下：
start get url: https://www.baidu.com
start get url: https://www.mi.com
start get url: https://www.qingyidai.com
轻易贷 - 开元金融旗下品牌_网络借贷信息中介服务平台
百度一下，你就知道
小米商城 - 小米9、小米MIX 3、红米Note 7，小米电视官方网站
``` 

当你执行这个脚本的时候，假如你在爬取的方法里面加一个sleep，这时候你在终端里面使用ps，你会看到4个进程，其中一个是父进程，其它3个是启动的子进程

感兴趣的可以再看看PHP的官方文档，上面提供了非常丰富的函数！https://www.php.net/manual/zh/book.pcntl.php 和 https://www.php.net/manual/zh/book.posix.php
