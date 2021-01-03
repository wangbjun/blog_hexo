---
title: Ubuntu Nautilus隐藏文件(夹)
date: 2020-03-15 18:57:06
category: Linux
tags: 
    - Ubuntu
    - Linux
---
如何在Ubuntu自带的文件夹管理器Nautilus里面隐藏文件或文件夹呢？

先说一下背景，之所以想到这个问题是因为我的电脑是 Win10+Ubuntu 双系统，有一块硬盘是共享的NTFS格式，在Windows下面分区会有2个文件夹，了解Windows的人应该知道，这是垃圾回收站和磁盘卷信息。

默认情况下在Windows里面这2个文件夹是隐藏的，但是当我切换到Ubuntu的时候，Ubuntu就会给显示出来了，这倒也正常，但是看起来很碍事，不爽，既然不爽我就要搞它。

<!--more--->

<img src = "/images/2020/2020-03-15_18-58.png" />

Nautilus这个文件管理器在很多Linux发行版里面都会使用，我觉得还挺好用哈，默认情况下，它不显示隐藏文件（以.开头的文件），但使用 Ctrl+H 快捷键可以快速显示隐藏文件。

现在问题来了，这2个是文件夹，而且也不是以.开头命名，咋搞呢？

不卖关子了，经过我简单查询，有一种比较简单且行之有效的方式：那就是新建一个.hidden的文件，里面写入你想隐藏的文件或文件夹名字就可以了。
```bash
jwang@jun:/media/jwang/Data$ cat .hidden 
$RECYCLE.BIN
System Volume Information
```
最后，经过我调研总结，Ubuntu的Nautilus文件管理器不显示以下3种文件：
- 隐藏文件，即文件名以 (.) 开头的文件。
- 备份文件，即文件名以 (~) 结尾的文件。
- 在指定文件夹中的 .hidden 文件里列出的文件。
   
如果有遇到类似问题的小伙伴可以参考一下了。
