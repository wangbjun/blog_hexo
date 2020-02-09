---
title: Linux系统自定义Shell命令
date: 2015-10-05 11:05:00
category: Linux
tags: 
    - Linux
---

### 一.应用场景
由于长期使用Ubuntu系统开发和日常生活，每天开机第一件事情就是更新系统，在Ubuntu系统下面更新往往需要好敲好几个命令：
```
sudo apt-get update		           #更新源
sudo apt-get upgrade		           #更新普通软件包
sudo apt-get dist-upgrade		   #更新系统软件包
sudo apt-get autoremove 		   #卸载无用的软件包
sudo apt-get autoclean			   #清除软件包缓存
```
通常情况下，我是一条接着一条敲，虽然看着命令行滚动很过瘾，但是时间长，感觉也没意思了，能不能用一条命令代替上面这些命令呢？
有人说，可以，你只要把这些命令行存起来，以后复制一下就搞定了...

<!--more-->

### 二.环境变量
其实我们是可以自定义命令的，其中关键点就在于环境变量，很多用windows系统的估计也知道环境变量这个东西，当初学Java的时候都知道在系统设置里面有个环境变量设置,
在path里面加一个路径，然后在cmd命令行下面敲java就有反应了，其实在Linux系统里面也是一样的。
```
jwang@jwang:echo $PATH
jwang@jwang:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/home/jwang/Bin
```
PATH是shell的全局变量，类似框架里面初始化的时候加载的一个超全局变量，里面存的就是当前用户的环境变量信息，一般情况下就是各种bin执行文件的路径。
  
比如最常见的/usr/bin是普通用户，/sbin是root用户特有，还有一些软件安装时候自动加进去的，比如那个/usr/games。

这个PATH路径的意义就是只要在是上面路径文件夹里面的可执行文件，就可以直接在shell里面执行，比如说 ifconfig，你们说ifconfig的可执行文件放在哪里呢？可以用whereis命令
```
jwang@jwang:whereis ifconfig
jwang@jwang:ifconfig: /sbin/ifconfig /usr/share/man/man8/ifconfig.8.gz
```
由上可知，ifconfig命令实际上是放在/sbin里面，这意味着你也可以这样用
```
jwang@jwang:/sbin/ifconfig
```
### 三.自定义命令
说到这里，估计有人就明白了，那是不是只要我把一个脚本放在PATH里面任意一个目录里面，然后我就可以直接敲，不用写全路径了？Yes，就是这样，比如说
```
jwang@jwang:~$ cd Bin/
jwang@jwang:~/Bin$ pwd
/home/jwang/Bin

jwang@jwang:~/Bin$ ls
update

jwang@jwang:~/Bin$ cat update
#!/bin/bash
sudo apt-get update
sudo apt-get -y upgrade
sudo apt-get -y dist-upgrade
sudo apt-get -y autoremove
sudo apt-get -y autoclean
jwang@jwang:~/Bin$
```
我在用户目录下新建一个Bin文件夹，里面放了一个update脚本，脚本里面内容就是系统更新那些命令，我可以这样做：
```
jwang@jwang:sudo ln -s /home/jwang/Bin/update /usr/bin/update
```
上面的命令是建立一个软链接到/usr/bin目录，这样就可以直接敲update命令了
```
jwang@jwang:~$ update
Hit:1 http://cn.archive.ubuntu.com/ubuntu xenial InRelease
Hit:2 http://cn.archive.ubuntu.com/ubuntu xenial-updates InRelease
.........
.........
Fetched 102 kB in 2s (42.0 kB/s)
Reading package lists... Done
Reading package lists... Done
Building dependency tree
Reading state information... Done
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Reading package lists... Done
Building dependency tree
Reading state information... Done
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Reading package lists... Done
Building dependency tree
Reading state information... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Reading package lists... Done
Building dependency tree
Reading state information... Done

```

是不是很方便呢？
还有另外一个方式，就是修改环境变量，把/home/jwang/Bin目录添加到环境变量里面，修改用户目录下的.bashrc文件，或者在全局文件/etc/profile添加一下语句
```
export PATH="$PATH:/home/jwang/Bin"
```
然后执行一下
```
source /etc/profile
```
当然，如果你只想临时修改一下环境变量，可以直接在命令行修改PATH的值，但是退出当前命令行就失效了
```
jwang@jwang:PATH=$PATH:/home/jwang/Bin
```
### 四.总结
以前我是一直没弄明白环境变量是什么意思，一直按着教程配置，前几天突然想明白了，环境变量说得通俗易懂点就是说明当前环境有哪些命令可以使用，实际上是在告诉那些程序, 
如果你找不到这个命令，你可以到这些目录里面找找，都找不到就报错！

日常生活工作中，可以把一些比较长的命令封装一下写个脚本，随便取个名字，只要名字不冲突就行（如果名字一样，
在PATH路径里面谁的目录在前面就优先执行谁），还是挺有用的。
