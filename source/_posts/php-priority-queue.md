---
title: PHP优先队列
date: 2017-06-05 15:00:00
category: PHP
tags: 
    - PHP
    - Queue
---

#### 1.什么是优先队列？

队列大家应该都很熟悉，专业的说队列是一种特殊的线性表，简单的说就是先进先出（FIFO），与队列相反的还有一种数据结构叫作栈，先进后出（FILO），这里的栈和内存里面的栈没啥关系，不要理解错了！

队列在开发的应用挺多的，最广泛的就是消息队列，用来处理一些任务比如下单，抢购，需要按请求的时间排序，先来的先处理，关键是保持一种顺序结构。实际开发中，我们一般很少自己去实现队列，通常都是使用一些现成的服务，比如redis queue，rabbitmq。

优先队列（Priprity Queue），顾名思义，就是带有优先级的队列，也就是说不是按请求的顺序排序，而且根据某一些规则属性。举个例子：有一些12306的刷票软件，花钱买了加速包抢到票的几率更高。这里所谓几率更高换个说法就是优先级更高，如果只有10张票，肯定是先让那些花了钱的先抢到票，没花钱的话排后面。

<!--more-->

#### 2.为什么需要优先队列？

假设现在有10000个人抢票，其中有50个人交了数目不一的钱，当系统抢到一张票后需要按照这些用户交钱的数目从大到小排序依次分配。如果让你去实现上面所说的抢票优先级，你会怎么设计呢？

##### 做法一：

如果这些用户信息是存储到数据库里面，当每次抢到一张票的时候，使用sql语句排序取出符合条件的用户里面交钱最多的那位就行了。如果不是存储到数据库里面的，可能就需要在内存里面排序了，1万个用户信息虽然不多，但是你每次都需要重新排序。


##### 做法二：

使用redis sorted set 实现，Redis 有序集合和集合一样也是string类型元素的集合,且不允许重复的成员。不同的是每个元素都会关联一个double类型的分数，redis正是通过分数来为集合中的成员进行从小到大的排序，有序集合的成员是唯一的,但分数(score)却可以重复。
```shell
sorted set 操作
ZADD：向 sorted set 中添加元素
ZCOUNT： sorted set 中 score 等于指定值的元素有多少个
ZSCORE：sorted set 中指定元素的 score 是多少
ZCARD： sorted set 中总共有多少个元素
ZREM：删除 sorted set 中的指定元素
ZREVRANGE：按照从大到小的顺序返回指定索引区间内的元素
ZRANGE: 按照从小到大的顺序返回指定索引区间内的元素
```
值得一说的是，这个并不是并发安全的，因为取优先级最高的元素以及删除这个元素是两次操作，不是原子性的，不过可以使用lua脚本解决这个问题。

##### 做法三：

使用优先队列，大部分编程语言的标准库里面都自带优先队列实现，并不需要自己去实现，不过像PHP这样的Web程序每次请求结束后内存数据都会被销毁，使用自己构建的优先队列还不如第二种做法好使，或者实现一个常驻进程的服务供Web调用。


#### 3.原理和使用

优先队列是基于二叉堆的，构建一个优先队列实际上就是在构建一个二叉堆，二叉堆是一种特殊的堆，二叉堆是完全二元树（二叉树）或者是近似完全二元树（二叉树）。

二叉堆有两种：最大堆和最小堆。最大堆：父结点的键值总是大于或等于任何一个子节点的键值；最小堆：父结点的键值总是小于或等于任何一个子节点的键值。

二叉树是每个结点最多有两个子树的树结构。

树是一种非线性的数据结构，是由n（n >=0）个结点组成的有限集合。

>以上内容仅供参考，关于这些数据结构的实现和算法细节这里不说了，毕竟不简单，感兴趣的话可以详细了解一下。

这些算法虽然不简单，但是毕竟我们都是站在巨人的肩膀上，下面看一下在PHP SPL里面提供的优先队列实现。PHP的标准库里面提供了常用的数据结构，比如链表，堆，栈，最大堆，最小堆，固定大小数组，其中就有优先队列，其类摘要如下：
```php
SplPriorityQueue implements Iterator , Countable {
    /* 方法 */
    public __construct ( void )
    public int compare ( mixed $priority1 , mixed $priority2 )
    public int count ( void )
    public mixed current ( void )
    public mixed extract ( void )
    public int getExtractFlags ( void )
    public void insert ( mixed $value , mixed $priority )
    public bool isCorrupted ( void )
    public bool isEmpty ( void )
    public mixed key ( void )
    public void next ( void )
    public void recoverFromCorruption ( void )
    public void rewind ( void )
    public void setExtractFlags ( int $flags )
    public mixed top ( void )
    public bool valid ( void )
}
```
其中常用的是compare，count，current，insert，next，rewind，valid等方法，用法也相对简单，下面看一个完整的例子：
```php
<?php
$queue = new SplPriorityQueue();

$queue->insert("A", 2);
$queue->insert("B", 17);
$queue->insert("C", 4);
$queue->insert("D", 10);
$queue->insert("E", 1);

//获取优先级最高的元素
echo $queue->top()."\n";

//按照优先级从大到小遍历所有元素
while ($queue->valid()) {
    echo $queue->current()."\n";
    $queue->next();
}
```

默认情况下，这个是按照数值大小排序的，但是如果排序比较的属性的并不是一个数值怎么办呢？比如说是对象，这时候可以采用下面的写法，我们可以新建一个类继承标准库的类，然后根据自己的规则重写compare的方法：
```php
<?php
class MyQueue extends SplPriorityQueue
{
    public function compare($priority1, $priority2)
    {
        if ($priority1->age === $priority2->age) {
            return 0;
        }
        return $priority1->age < $priority2->age ? -1 : 1;
    }
}

class Person
{
    public $age;
    public function __construct($age)
    {
        $this->age = $age;
    }
}

$queue = new MyQueue();

$queue->insert("A", new Person(2));
$queue->insert("B", new Person(17));
$queue->insert("C", new Person(4));
$queue->insert("D", new Person(10));
$queue->insert("E", new Person(1));

//获取优先级最高的元素
echo $queue->top() . "\n";

//按照优先级从大到小遍历所有元素
while ($queue->valid()) {
    echo $queue->current() . "\n";
    $queue->next();
}
```

大家看懂了吗？如果错误欢迎指正！





