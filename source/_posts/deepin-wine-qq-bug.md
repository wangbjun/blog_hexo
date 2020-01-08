---
title: 解决Deepin-Wine-QQ或微信图标Bug
date: 2018-12-01 12:11:08
category: 工具
tags: 
    - Deepin-Wine
---

之前写过一篇文章说在Linux下面使用deepin的wine QQ和微信, 虽然这个版本挺好用，但是一直以来有个bug困扰我：QQ和微信的图标都是wine的小图标，一模一样不说，还重叠在一起，当你使用 **ctrl+tab** 切换应用的时候很头疼，用过的人应该生有感受！

![image](http://upload-images.jianshu.io/upload_images/3571187-9b36a691ddfe0a8b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<!--more-->

有段时间我网上查了很久都没有找到答案，起初以为是图标问题！在Linux下面桌面图标快捷方式是由一个desktop文件配置，比如微信的内容基本上如下：
```bash
#!/usr/bin/env xdg-open

[Desktop Entry]
Encoding=UTF-8
Type=Application
X-Created-By=Deepin WINE Team
Categories=chat;
Icon=deepin.com.wechat
Exec="/opt/deepinwine/apps/Deepin-WeChat/run.sh" -u %u
Name=WeChat
Name[zh_CN]=微信
Comment=Tencent WeChat Client on Deepin Wine
StartupWMClass=WeChat.exe
MimeType=
```
其中有几个比较关键的地方，一个是Icon，一个是Exec，还有Name，有一天我看到这个 **StartupWMClass** 突发奇想，虽然我不懂是啥意思，但是感觉这个有问题。

于是百度了一下，基本上找不到任何内容，只有一篇文章，点进去居然是404...还好有百度快照！

![image](http://upload-images.jianshu.io/upload_images/3571187-65b3cf68d42da7b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

终于找到问题所在了，默认情况下，Linux是根据可执行文件的名称判定是属于哪个desktop文件配置的，大部分desktop文件的Exec配置的可执行文件刚好就是实际执行的文件名，所以很多没有StartupWMClass配置项。

但是这个配置项很重要，比如说在上面的微信的配置里面这个值是”WeChat.exe“，但是为什么还是不行呢？

根据文章的说法，可以通过```xprop WM_CLASS```获取窗口的属性值，在命令行下执行这个命令，鼠标会变成+，然后点击要QQ或微信的窗口：
```bash
jwang@jwang:~$ xprop WM_CLASS
WM_CLASS(STRING) = "wechat.exe", "Wine"
```
不知道这个值是不是不同的电脑不一样，反正在我的电脑上面这个值是”wechat.exe“，居然是小写！

这样的话，我们只需把desktop配置文件里面的StartupWMClass改成小写的就行了，问题解决！
