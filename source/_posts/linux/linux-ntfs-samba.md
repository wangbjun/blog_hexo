---
title: Linux下挂载NTFS硬盘和Samba共享
date: 2018-12-02 11:05:00
category: Linux
tags: 
    - Linux
    - Samba
---

###  1.挂载 NTFS 硬盘
讲道理是不建议在Linux下面使用ntfs这种文件系统，Linux有个专用的文件系统ext4，但是为什么这么用呢？主要原因还是为了兼容Windows，ntfs是Windows最常用的文件系统。

还有一种情况是双系统，为了能在Linux和Windows下面都能读取到，只能使用ntfs格式，毕竟Linux对ntfs格式还算是挺友好的，但是Windows对ext4貌似不是那么友好，虽然也有软件能读取，但是麻烦!

<!--more-->

默认情况下，主流Linux发行版是支持ntfs格式的分区的，如果不支持的话需要安装一个软件就行：
```
sudo apt install ntfs-3g
```
然后你在文件管理的右边就会看到可以挂载的分区，其实这时候还没有挂载，鼠标点一下会自动挂载，下面里面的Data和Video分区就是我挂载好的：

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1fvyhm4x86mj20580bgaa7.jpg)

如果你用的不是桌面发行版，可以使用 mount 命令挂载

问题来了，为了方便，需要实现每次开机自动挂载，这需要修改一个配置，Linux的磁盘挂载配置在 /etc/fstab 文件，你可以手动编写这个配置，这里给一个示例：
```
# /etc/fstab: static file system information.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>

#Entry for /dev/nvme0n1p5 :
UUID=ccaace56-1c45-487c-ac0b-b337c37c107f	/	ext4	errors=remount-ro	0	1
#Entry for /dev/nvme0n1p1 :
UUID=257D-EDE3	/boot/efi	vfat	defaults	0	1
#Entry for /dev/sda1 :
UUID=5C5AAFB95AAF8E78	/media/jwang/Data	ntfs-3g	defaults,nodev,nosuid,locale=en_US.UTF-8	0	0
#Entry for /dev/sda2 :
UUID=4274B7A774B79C5B	/media/jwang/Video	ntfs-3g	defaults,nodev,nosuid,locale=en_US.UTF-8	0	0
#Entry for /dev/nvme0n1p2 :
UUID=13b7dfee-a639-464f-b2f5-c7b2e435b71d	none	swap	sw	0	0

#UUID=94A4-85E8	/boot/efi	vfat	umask=0077	0	1
```
需要注意的是，这里面有些分区是安装系统的时候自动挂载上去的。这里说一个小bug，如果你这个配置文件不对，每次开机的时候就会卡很久，大概30s左右。

因为这个配置文件是在开机的时候自动执行的，如果系统找不到你配置的磁盘或者挂载点，就会一直等，最后超时就会跳过。

所以如果你哪天发现你开机的时候很慢,不妨看看这个文件。

不过呢，手动配置还是比较麻烦，这里建议大家使用一个软件去配置挂载ntfs分区，名字叫 ntfs-config：
```
sudo apt install ntfs-config
sudo ntfs-config
```
会弹出一个图形界面，配置一下即可，简单方便，如果不是桌面版的话，你需要好好研究研究这个fstab文件的了，其实也不难，看看官方文档就可以了,这里不细说了！

###  2.Samba共享
这个其实挺实用的，很多路由器，nas都是用的这个共享文件，简单的说SMB是一种文件共享协议，Samba这个软件实现了这种协议，厉害的地方就在于SMB这个协议被Windows，Mac，Android等很多操作系统都支持。

你可以很方便的把你电脑上的文件通过网络(一般都是局域网)共享给别人。举个例子，在公司共享文件给同事，在家里可以共享一下电脑上的电影，照片啊，手机上装一个文件浏览器也可以。

个人经常用到的是把电脑下载好的电影共享出来，Android手机上安装一个叫作ES文件浏览器的App就可以直接“在线”看电影了。

```
sudo apt install samba
```
打开一个文件夹的属性你就会看到共享的选项，可以选择只读或者读写，也可以设置访问账号和密码，如果只是自己用，全部都勾上就行了！

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1fvyhy77p2qj20ek0f9q3x.jpg)

就是这么简单，但是如果你不是用的桌面版Linux，那也没问题，Samba的配置文件位于 /etc/samba/smb.conf, 自己加一个配置就行，配置文件示例：
```
[profiles]
    comment = Users profiles
    path = /home/samba/profiles
    guest ok = no
    browseable = no
    create mask = 0600
    directory mask = 0700
```

> 这里说一个疑难杂症，有可能有人遇到过，就是挂载的ntfs分区使用Samba共享的时候可能会出现共享权限问题，就是对方可以看到共享的文件夹，但是点击文件夹提示没有权限。
排除了文件夹权限之后，我最后找到一个解决方案，就是在Samba的配置文件里面加入一个配置: force user = your-user-name
