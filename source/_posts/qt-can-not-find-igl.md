---
title: Qt-运行-cant't-find-lGL
date: 2018-09-03 13:22:00
category: Linux
tags: 
    - QT
---

**实验问题：运行最简单”hello world!”,出现can’t find -lGL的问题**

**实验阵地： ubuntu14.04+qt5.2**

**问题分析**： 
出现该类问题的原因有2个： 

(1)没有安装libGL; 

(2)libGL没有正确链接。

<!--more-->

**问题解答**： 

（1）如果是问题1,这个好办。只要安装libGL即可。这个在其他博客中也都有提到, 如[http://blog.sina.com.cn/s/blog_500bd63c0102uzmt.html](http://blog.sina.com.cn/s/blog_500bd63c0102uzmt.html) 
只需终端执行

```
$ sudo apt-get install build-essential 
$ sudo apt-get install libgl1-mesa-dev
```

安装libGL即可。（libGL是openGL的库） 

（2）如果是问题2,就稍微难办一点。 
首先，我们利用命令

```
$ /sbin/ldconfig -v | grep GL
```

查看所有有关GL的链接库的链接关系。 
如果是问题2，则会有这样的打印信息

```
/sbin/ldconfig.real: Cannot stat /usr/lib/x86_64-linux-gnu/mesa/libGL.so: No such file or directory
```

表示”无法获取libGL的链接信息：没有该文件或目录”。我们进入/usr/lib/x86_64-linux-gnu/mesa/

```
$ cd /usr/lib/x86_64-linux-gnu/mesa/
```

确实能找到libGL.so。但因为不存在与之相关的硬链接，而导致libGL.so失效。 
这时候，应该怎么办呢？ 

a)首先我们进一步确认一下libGL.so是否失效。（毕竟之后涉及到在/usr/lib/x86_64-linux-gnu文件夹下删除，一不小心删错了，可是要命的）

```
$ ls -l libGL.so 
```

查看libGL的硬链接，如果libGL存在硬链接的话，会出现类似信息：

```
lrwxrwxrwx 1 root root 13 12月  4 20:42 libGL.so -> ../libGL.so.1
```

如果出现

```
0 libGL.so
```

或其他错误信息，则说明这个libGL.so已经失效。 

b)之后，搜索是否存在libGL.so的硬链接。（一般如果第一步，安装已经做过的话，是肯定存在的）

```
$ cd 
$ sudo find /usr/lib/ -name libGL.so*
```

打印信息

```
/usr/lib/x86_64-linux-gnu/mesa/libGL.so
/usr/lib/x86_64-linux-gnu/libGL.so.1.0.0
/usr/lib/x86_64-linux-gnu/libGL.so
/usr/lib/x86_64-linux-gnu/libGL.so.1
```

我们发现在/usr/lib/x86_64-linux-gnu/文件夹下存在硬链接libGL.so.1.0.0 
接下来，我们的问题就只剩下如何让/usr/lib/x86_64-linux-gnu/mesa/libGL.so关联上/usr/lib/x86_64-linux-gnu/libGL.so.1.0.0 

由于在/usr/lib/x86_64-linux-gnu/中libGL.so.1是libGL.so.1.0.0的软链接，所以我们只要将/usr/lib/x86_64-linux-gnu/mesa/libGL.so关联上/usr/lib/x86_64-linux-gnu/libGL.so.1即可 

执行以下操作:

```
$ cd /usr/lib/x86_64-linux-gnu/mesa/
$ sudo rm libGL.so #删除libGL.so
$ sudo ln -s ../libGL.so.1 libGL.so #创建软链接
```

重新运行

```
ls -l libGL.so 
```

这时应该会有打印信息

```
lrwxrwxrwx 1 root root 13 12月  4 20:42 libGL.so -> ../libGL.so.1
```

再次运行

```
$/sbin/ldconfig -v | grep GL
```



```
/sbin/ldconfig.real: Cannot stat /usr/lib/x86_64-linux-gnu/mesa/libGL.so: No such file or directory
```
上述错误会消失。 

重新编译qt，编译成功！
