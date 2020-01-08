---
title: PHP数组解析和常见操作
date: 2016-02-05 19:00:03
category: 编程开发
tags: 
    - PHP
    - 数组
---

在PHP里面使用最多的数据结构恐怕就是数组了，不过PHP的数组和我们传统意义上的数组区别很大，PHP的数组功能上相当于其它语言里面array+list+map数据结构的集合体，这就是动态语言的强大之处。在PHP里面有2种数组，一种是传统的索引数组，另一种是关联数组，其实就是其它语言里面map数据结构。

#### 底层实现
PHP的数组底层是使用HashTable实现，说到哈希表估计很多人都了解过，PHP数组通过一个映射函数把key映射到对于的value值上面，所以查找起来非常快，时间复杂度是O(1),哈希表都会遇到冲突问题，在PHP里面是通过链表的方式解决的。
```
//Bucket：散列表中存储
typedef struct _Bucket {
	zval              val;  //存储的具体value，这里嵌入了一个zval，而不是一个指针
	zend_ulong        h;    //key根据times 33计算得到的哈希值，或者是数值索引编号
	zend_string      *key;  //存储元素的key
} Bucket;
 
//HashTable结构
typedef struct _zend_array HashTable;
 
struct _zend_array {
	zend_refcounted_h gc;
	union {
		struct {
			ZEND_ENDIAN_LOHI_4(
				zend_uchar    flags,
				zend_uchar    nApplyCount,
				zend_uchar    nIteratorsCount,
				zend_uchar    reserve)
		} v;
		uint32_t flags;
	} u;
	uint32_t          nTableMask;      //哈希值计算掩码，等于nTableSize的负值(nTableMask = -nTableSize)
	Bucket            *arData;         //存储元素数组，指向第一个Bucket
	uint32_t          nNumUsed;        //已用Bucket数
	uint32_t          nNumOfElements;  //哈希表有效元素数
	uint32_t          nTableSize;      //哈希表总大小，为2的n次方
	uint32_t          nInternalPointer;
	zend_long         nNextFreeElement;  ////下一个可用的数值索引,如:arr[] = 1;arr["a"] = 2;arr[] = 3;则nNextFreeElement = 2;
	dtor_func_t       pDestructor;

```

<!--more-->

PHP7源码里面具体涉及到结构体如上，源码我就不解读了，主要是我也不太熟悉，只是看过一些介绍文章，但是希望大家可以了解一下，下面我主要介绍一下PHP数组的一些常用函数，回顾一下基础。

#### 常用函数
PHP的数组函数非常多，但是说起这点我就头疼，PHP的数组函数命名有些非常奇葩，有以**array_**开头的,也有一些不知道根据啥命名的...下面我就分类介绍一下：

##### 1.排序类
```
// 默认排序是按从低到高，而且是引用传递，第二个参数可以选择排序类型
sort ( array &$array [, int $sort_flags = SORT_REGULAR ] ) : bool
```
然后是一些以a、r、k、u组合的函数，不得不说这命名是真烂！

```
a 是associate，意思是排序是保留索引关联，最常见的是 asort
r 是reverse，意思是逆序排，最常见的就是 rsort
k 是key，意思是按照数组的key进行排序，保留索引关联，主要是用于关联数组，最常见的就是 ksort
u 是user，意思使用用户自定义函数的函数排序，最常见的就是 usort
```

好了，除了上面这4个之外，其它就是这几个字母的组合的函数了，比如 **arsort** 是保留索引倒序排序、**uksort** 使用用户自定义的比较函数对数组中的键名进行排序，其它我就不多说了。

##### 2.遍历类
除了可以使用for 和 foreach循环遍历数组之外，PHP还有很多其它遍历数组，并且操作数组的函数
```
//为数组的每个元素应用回调函数
array_map ( callable $callback , array $array1 [, array $... ] ) : array

//使用用户自定义函数对数组中的每个元素做回调处理

array_walk ( array &$array , callable $callback [, mixed $userdata = NULL ] ) : bool

array_walk_recursive — 对数组中的每个成员递归地应用用户函数

array_reduce — 用回调函数迭代地将数组简化为单一的值

array_replace_recursive — 使用传递的数组递归替换第一个数组的元素
```
##### 3.其它
```
array_flip — 交换数组中的键和值
array_reverse — 返回单元顺序相反的数组
array_column — 返回数组中指定的一列
array_combine — 创建一个数组，用一个数组的值作为其键名，另一个数组的值作为其值

array_diff — 计算数组的差集
array_intersect — 计算数组的交集

array_filter — 用回调函数过滤数组中的单元
array_flip — 交换数组中的键和值

array_keys — 返回数组中部分的或所有的键名
array_values — 返回数组中所有的值

array_rand — 从数组中随机取出一个或多个单元
shuffle — 打乱数组

array_product — 计算数组中所有值的乘积
array_sum — 对数组中所有值求和

array_search — 在数组中搜索给定的值，如果成功则返回首个相应的键名
array_key_exists — 检查数组里是否有指定的键名或索引
in_array — 检查数组中是否存在某个值

array_replace — 使用传递的数组替换第一个数组的元素
array_slice — 从数组中取出一段
array_splice — 去掉数组中的某一部分并用其它值取代
```

PHP自带的这些数组函数基本上你所想到的操作它都有，没有的也可以组合这些方法创造一个，我记得在laravel框架里面就自带了一个数组集合类，里面就有一些非常好用的方法。

有人问，这么多函数，怎么能记住？

其实我觉得大部分时候并不要死记硬背，面试除外，当你遇到问题的时候至少心里有点数，具体参数可以查下文档，或者使用IDE的联想功能，平时没事多看看官方文档也挺好的。





