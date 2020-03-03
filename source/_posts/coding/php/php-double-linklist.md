---
title: PHP数据结构之双向链表
date: 2015-12-01 11:01:13
category: 编程开发
tags: 
    - PHP
---

其实这些都是PHP SPL 标准库里面的东西, SPL是用于解决典型问题(standard problems)的一组接口与类的集合。说白了，这是PHP官方实现的一些数据结构,
印象中Java的标准库就很强大，不要慌，PHP也有。

![](https://ww1.sinaimg.cn/large/5f6e3e27ly1fv0rjzmm26j20hz0himyz.jpg)

按顺序来，先讲一下这个双向链表(double link list)，数据结构讲的是思想，不分编程语言，所以先回顾一下基本概念吧。

<!--more-->

> 链表是一种物理存储单元上非连续、非顺序的存储结构，数据元素的逻辑顺序是通过链表中的指针链接次序实现的。链表由一系列结点（链表中每一个元素称为结点）组成，
结点可以在运行时动态生成。每个结点包括两个部分：一个是存储数据元素的数据域，另一个是存储下一个结点地址的指针域, 相比于线性表顺序结构，操作复杂。

这是百度百科的介绍，不黑不吹，讲的还是非常准确的。我觉得链表最大的特点就是 非连续和非顺序存储，和其对应的就是数组，大家都知道数组在内存里面是连续存储的, 
由于是连续存储，操作系统在每次分配内存的时候并不一定刚好有那么大小的一块连续的内存，于是就会产生内存碎片。而且数组还有一点不好，想找一个数得从头开始一个个找,
其查找时间是O(n),链表是O(1)。总的来说，链表是为了解决数组的不足。

有一点需要说一下，PHP的数组并不是传统意义的数组，在C和Go等语言里面，数组是一个固定大小，固定类型的数据集合，但是PHP的数组啥都干,其功能应该是集了数组，切片，链表，map等数据结构的综合体,
在很多其它编程语言里面，这些数据结构是分的非常清楚。所以有时候理解这些概念的时候，不要拿PHP的数组对号入座哈。

借2张图理解一下：

![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1536298900470&di=bed31d47583655de7d4e623e3c4d2bdf&imgtype=jpg&src=http%3A%2F%2Fimg3.imgtn.bdimg.com%2Fit%2Fu%3D1355954038%2C3984007412%26fm%3D214%26gp%3D0.jpg)

![](https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=152820947,1698821981&fm=26&gp=0.jpg)

举个形象的例子，这个链表就有点像链条，每一个链条单元是首尾相接，自行车链条就是环型链表，而数组就是铁轨，直的，虽然跑的快, 但是要求高！

如果用PHP去实现链表倒是不难，毕竟PHP这么强大，可以用数组模拟，但是性能并不高，这里就看一下官方的实现吧：

![](https://ww1.sinaimg.cn/large/5f6e3e27ly1fv0sjz2p5qj20ke0mzdis.jpg)

可以看到这个类实现了Iterator，arrayAccess等接口，就意味着可以像数组一样访问这个对象，有push,pop,shift,unshift,current等方法。

举个例子：
```php
<?php
$dll = new SplDoublyLinkedList();
$dll->add(0, 'a');
$dll->add(1, 'b');
$dll->add(2, 'c');
$dll->add(3, 'd');
$dll->add(4, 'e');

var_dump($dll);
var_dump($dll->pop()); # 右边出列
var_dump($dll->shift()); # 左边出列

var_dump($dll->bottom()); # 第一个节点
var_dump($dll->top());    # 最后一个节点

$dll->unshift('b'); # 左边入列
$dll->push('d'); # 右边入列

//数组遍历
foreach ($dll as $value) {
    var_dump($value);
}

$dll->push('f');

$dll->setIteratorMode(SplDoublyLinkedList::IT_MODE_FIFO); # FIFO first insert first out

//循环遍历
for ($dll->rewind(); $dll->valid(); $dll->next()) {
    var_dump($dll->current()) . "\n";
}

```

这里还有一个 setIteratorMode 函数用于设置迭代器模式，它有4种模式，LIFO，FIFO，DELETE，KEEP。LIFO,是last in first out，即后进先出,
这种模式其实就是栈模式，栈是一种很常见的数据结构，最广泛的用途莫过于函数调用栈了。FIFO是队列模式，先进先出。DELETE是删除模式，KEEP是遍历模式，
默认是FIFO模式。

可见PHP这个双向链表还可以当栈和队列使用，没错！其实后面的 SplStack 和 SplQueue 就是继承的 SplDoublyLinkedList，完全一模一样，就是改个名字而已！

最后谈谈应用，由于PHP用来写web应用，每个请求完了就销毁了，很多设计模式和数据结构基本上很难用到，除非拿来写一些常驻后台应用。
比如说队列，一般都是用redis队列，或者rabbitmq等专业软件实现。但是你如果问可以用PHP来实现一个队列服务常驻后台吗？那是肯定可以的，但是意义不大,
PHP的运行效率和C等静态语言那比是差了10万八千里...so,多了解了解一些也是不错的，虽然不一定用得上
