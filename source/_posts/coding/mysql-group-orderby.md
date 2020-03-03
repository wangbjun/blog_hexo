---
title: 解决MySQL分组排序求最值问题
date: 2018-01-05 19:00:03
category: 编程开发
tags: 
    - MySQL
    - 数据库
---

首先，先明确一下问题，所谓求分组的最值意思的就是在sql里面使用group by之后，每个分组有多条数据，我们要根据一定条件取其中最大的一条或者多条！

先看一个数据表 blogs 结构，简单说一下，cat_id 就是分类ID，可以看到一个分类有多条记录：

![image](https://upload-images.jianshu.io/upload_images/3571187-8d13727754425021.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

举个非常典型的问题:

1.求某个分类ID下，查看次数最多的3条数据？这个问题很简单，基本上大家都能写出来这样的sql:

<!--more-->

```sql
select * from blogs where cat_id = $cat_id order by view_num desc limit 3;
```
2.求多个分类ID下，查看次数最多的3条数据呢？这个问题就在于求多个，也就是我要批量查询，不能一个个查，有很多人图省事就直接for循环一个个查了,如果说只有几个ID这样做还可以，如果有几十个这样的数据就意味着几十次的查库操作，对性能影响还是挺大的，所以必须想办法！

sql如下：

```sql
select SUBSTRING_INDEX(GROUP_CONCAT(cat_id,'-',id ORDER BY view_num),',',3) from blogs where cat_id in(1,2,3,4) GROUP BY cat_id
```
这条语句看上去比较复杂，不要慌，SUBSTRING_INDEX 是内置函数，功能类似于PHP里面的 substr，在这意思是取前3个数据，重点是 group_concat, 这个函数很多人都用过，但是我估计很多人都不知道后面还可以写 order by，所以这条sql的意思就是在每个分组里面排序取前3个。

但是取出来的数组格式并不好看，是以 cat_id-id 这种形式取出来的，可以看到有多个：

![image](https://upload-images.jianshu.io/upload_images/3571187-f5ba49d28a69e01a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

后面的操作只能拿到代码里面处理了，可能需要循环取出所有id，然后批量获取数据，最后再拼接出来想要的数据！虽然比较麻烦，在代码里面需要多出很多次for循环操作，但是相比于多查几十次库，这点代码运行开销还是很小的！
