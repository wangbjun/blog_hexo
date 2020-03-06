---
title: 闲谈PHP面向对象编程
date: 2016-03-09 15:00:00
category: 编程开发
tags: 
    - PHP
    - OOP
---

关于面向过程和面向对象编程之间的区别这里不多说,简单看了一个例子,如何解决把大象装进冰箱这个问题?
### 面向过程方案:
第一步.打开冰箱

第二步.把大象放进冰箱

第三步,关上冰箱

这个方案看上去没什么毛病,简单明了,当然第二步可以拆分的更细,拆成更多小步骤!用代码简单演示如下:
```
<?php
//1.打开冰箱
function openFridge($fridge)
{
    echo "打开冰箱:" . $fridge['name'];
}
//2.放置大象
function placeElephant($elephant, $fridge)
{
    if ($elephant['weight'] > $fridge['width']) {
        echo "大象太大了!";
        return false;
    }
    echo "把{$elephant['name']}放进去:" . $fridge['name'];
    return true;
}
//3.关上冰箱
function closeFridge($fridge)
{
    echo "关掉冰箱:" . $fridge['name'];
}
```
然后实际操作的代码:
```
$fridge = [
    'name'   => '西门子冰箱',
    'width'  => 20,
    'height' => 50,
];
$elephant = [
    'name'   => '非洲大象',
    'weight' => 100,
];
//三步走
openFridge($fridge);
placeElephant($elephant, $fridge);
closeFridge($fridge);
```
面向过程编程典型的做法就是定义一大堆函数,一个函数干一件事,然后依次调用各个函数完成一个功能.优点也有,比如简单省事,缺点也很多,比如项目大了之后维护是噩梦!

### 面向对象方案:
首先,得有2个类,一个是大象类:
```
//大象类
class Elephant
{
    private $name;
    private $weight;
    public function __construct($name, $weight)
    {
        $this->name     = $name;
        $this->weight   = $weight;
    }
    ....
}
```
另一个是冰箱类:
```
class Fridge
{
    private $name;
    private $width;
    public function __construct($name, $width)
    {
        $this->name   = $name;
        $this->width  = $width;
    }
    public function open()
    {
        echo "打开{$this->name}冰箱";
    }
    public function close()
    {
        echo "关闭{$this->name}冰箱";
    }
    public function store($something)
    {
        echo "放置{$something}";
    }
}
```
实际操作的时候代码:
```
$e = new Elephant('非洲大象', 12);
$f = new Fridge('西门子冰箱', 25);
$f->open();
$e->store($e);
$f->close();
```
当然这里还有一点小疑问,那就是大象放进冰箱这个操作是属于哪个对象?是冰箱的功能,还是大象的功能?换句话说是大象自己走进冰箱,还是大象被放进冰箱呢?很多时候,这2种方案都能实现你所需要的功能,那就得结合实际情况分析了!

换一种思路,假如说,我们定义冰箱有一个功能就是放东西,但是这个东西必须满足一定条件,比如说让对象自己选择采用何种方式放进冰箱等等,这时候我们可以定义一个接口:
```
interface Fridgeable
{
    public function intoFridge();
}
```
然后让需要放进冰箱的对象实现这个接口:
```
class Elephant implements Fridgeable
{
    ....
    //实现接口
    public function intoFridge()
    {
         echo "大象飞进了冰箱";
    }
}
```
这时候冰箱类就可以这样写:
```
...
public function store(Fridgeable $fridgeable)
{
    $this->open();
    $fridgeable->intoFridge();
    $this->close();
}
...
```
这时候操作就变成:
```
$e = new Elephant('非洲大象', 12);
$f = new Fridge('西门子冰箱', 25);
$f->store($e);
```
假如这时候还有一直兔子也要放进冰箱,我们只需要让这个兔子也实现这个接口即可:
```
class rabbit implements Fridgeable
{
    ....
    //实现接口
    public function intoFridge()
    {
         echo "兔子钻进了冰箱";
    }
}
```
而这种设计思想就叫依赖注入,又叫控制反转.冰箱只要定义好一个接口,所有想放进冰箱的对象只要实现这个接口就可以了,这样我们就不用在冰箱里面写一大堆代码,实现了解耦和扩展性!

好啦,就说这么多了,如果不恰当的地方请指正哈!
