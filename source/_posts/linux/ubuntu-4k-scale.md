---
title: Ubuntu 4K显示器缩放设置
date: 2019-05-03 12:21:06
category: Linux
tags: 
    - Ubuntu
---

![](https://wangbjun.github.io/images/16d0f0f03263cb7b.jpg)

开头一张图介绍一下我现在日常生活和开发使用的电脑配置：Ubuntu 16.04 + i7-8700k + 1060 + nvme ssd + 32G RAM + 4k显示器，这个配置倒不算很高端，但是开发用体验很高，系统的流畅程度非常高

<!--more-->

电脑CPU和内存可以差点，ssd是必须有的，另外还有一个亮点是LG的4k显示器，这个体验非常棒，现在4k显示器非常便宜，我这个也就2k左右的价格。

今天的主题就是4k显示器，众所周知，Mac的显示效果之所以出众是由于其高超的屏幕分辨率，几年前Mac都已经用上了3k分辨率，而且大多数Windows笔记本还用着1080p，苹果的IMac早已经用上了5k显示器。

换句话说，买Mac买的就是显示屏，没有屏幕的硬件加持，什么操作系统优化都是扯淡！有了4k显示器，你发现装上Windows显示效果也不差，不过这块我需要说一下，同等硬件下，Linux和Mac的显示效果确实要比Windows好一点，对高分屏的支持好很多。

## 版本
我个人比较喜欢unity桌面，所以还是用Ubuntu 16.04，我曾经尝试过Ubuntu 18.04，但是感觉gnome桌面在流畅度和易用性方面和unity还是有不少差距，所以本篇文章可能支持适合unity桌面吧

## 硬件
如果你要换4k显示屏，有一点需要注意，不少i7 CPU 内置集显理论上是带的动4k+60fps的，但是只支持dp接口，不支持hdmi，这一点可以在intel官网的cpu详细规格里面可以查阅。但是大部分主板都不会带dp接口，很少很少，只有极少部分高端主板会带，而现在大部分独显都会带dp口。

众所周知，NVIDIA的独显在Linux上面的驱动支持都不是太好，但是intel的集显支持非常好，如果你想要使用4k显示器，一个独显少不了，不过据我目前的使用体验来说，1060 表现还不错，建议大家开启高性能模式，如下图：
![](https://wangbjun.github.io/images/16d0f203bbf9601b.jpg)

## 缩放设置
这是重点，根据我经验，在4k+27英寸显示器的配置下，缩放设置很简单，不需要什么环境变量，直接在显示里面设置缩放就行，默认是1，设置一个1.75-2比较合适。
![](https://wangbjun.github.io/images/16d0f23300cd21d1.jpg)
实际上，上面这个设置好，已经可以解决99%的缩放问题了，不需要什么环境变量，上一些应用的图给大家看看：

![](https://wangbjun.github.io/images/16d0f26799e5bf08.jpg)

![](https://wangbjun.github.io/images/16d0f27645283d4e.jpg)

![](https://wangbjun.github.io/images/16d0f27f294742ce.jpg)

## deepin缩放
有些软件不走上面的缩放设置，比如deepin qq或wechat，估计很多用Linux的都会使用移植的deepin应用，但是也有办法:
```
WINEPREFIX=~/.deepinwine/Deepin-WeChat deepin-wine winecfg
```
在弹出的对话框里面找到graphics设置，设置一个比较合适的dpi，以我个人经验，150-170比较合适，如下图：
![](https://wangbjun.github.io/images/16d0f2ba8e833515.jpg)

## 网易云音乐
网易云的软件在4k下面也是个刺头，暂时没有完美的方案，但是有一个可以凑合用，在网易的desktop文件Exec配置里面加入：
```
--force-device-scale-factor=1.75
```
## 搜狗输入法
搜狗输入法其实也是不支持4k自动缩放的，不过我们可以把皮肤的字体设置大一点，达到的效果是一样的：
![](https://wangbjun.github.io/images/16d0f301af11cca2.jpg)

## QT系列软件
Linux下面有很多基于QT开发的软件，它的缩放有可能是跟系统走的，但是也有可能不是，根据我的经验，QT需要设置一下2个环境变量：
```
export QT_DEVICE_PIXEL_RATIO=2
export QT_AUTO_SCREEN_SCALE_FACTOR=1
```
为什么有2个呢？据说第一个是老版本会用到，但是这个缩放因子只支持整数倍，你不能写1.5，有点蛋疼！

有些软件设置之后可能会放的太大，这时候我建议针对不同的软件，可以在启动之前使用export设置环境变量，或者在其快捷方式里面设置一下都可以，特殊情况特殊处理，但是大部分设置为2的话还可以接受。
