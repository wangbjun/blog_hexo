---
title: 解决Linux下鼠标灵敏度问题
date: 2021-06-07 22:04:02
category: Linux
tags: 
    - Ubuntu
---

这可能不是一个普遍的问题，但是我遇到很多次，不知道经常使用Windows、Linux双系统的人有没有感觉到，那就是同样一个鼠标，DPI一样的情况下，在Linux上面比较“飘”，我主要是用Ubuntu，这个问题从14.04到18.04版本一直存在，鼠标手感比Windows差太多，也不是鼠标的问题，我期间换过好几个鼠标都一样。

具体的说，有2个方面，一个是鼠标移动加速度，通常情况下Linux下面鼠标移动非常快，就是感觉不准；另一个则是滚轮的滚动速度，在浏览网页的情况下翻页滚动非常慢。

说实话，Linux系统在这方面调教还是差了点，本文说说如何自己动手解决这些小问题，可以留作备用，以备不时之需。

<!--more-->

## 1.鼠标不准
Ubuntu的桌面系统对于鼠标有一些设置选项，比如双击的速度、鼠标指针移动速度，但是效果不明显，有时候即使把移动速度调到最低还是很快。

<img src="/images/2021/2021-06-07_22-12.png" /> 

你不能说Linux系统本身对鼠标的支持差，只不过很多参数属性并没有暴露出来给用户自定义，而默认的参数调教的也差强人意，但是我们可以通过一些命令去设置，毕竟Linux是个开源开放的系统。

在Linux里面我们可以使用```xinput list```查看电脑所有的输入设备信息：
```
⎡ Virtual core pointer                    	id=2	[master pointer  (3)]
⎜   ↳ Virtual core XTEST pointer              	id=4	[slave  pointer  (2)]
⎜   ↳ Razer Razer DeathAdder Essential        	id=10	[slave  pointer  (2)]
⎜   ↳ Razer Razer DeathAdder Essential Consumer Control	id=12	[slave  pointer  (2)]
⎣ Virtual core keyboard                   	id=3	[master keyboard (2)]
    ↳ Virtual core XTEST keyboard             	id=5	[slave  keyboard (3)]
    ↳ Power Button                            	id=6	[slave  keyboard (3)]
    ↳ Video Bus                               	id=7	[slave  keyboard (3)]
...
...
...
```
当我们需要查看某个设备的属性的时候，可以使用```xinput list-props id```,后面的id就是设备的id，在我的电脑鼠标就是10，不同电脑的值不一样，甚至同一个电脑插在不同的USB口也不一样，这一点需要注意一下。
```
jwang@jun:~$ xinput list-props 10
Device 'Razer Razer DeathAdder Essential':
	Device Enabled (168):	1
	Coordinate Transformation Matrix (170):	1.000000, 0.000000, 0.000000, 0.000000, 1.000000, 0.000000, 0.000000, 0.000000, 1.000000
	Device Accel Profile (293):	0
	Device Accel Constant Deceleration (294):	1.000000
	Device Accel Adaptive Deceleration (295):	1.000000
	Device Accel Velocity Scaling (296):	1.000000
	Device Product ID (286):	5426, 110
	Device Node (287):	"/dev/input/event4"
	Evdev Axis Inversion (297):	0, 0
	Evdev Axes Swap (299):	0
...
...
...
```
这上面的参数比较多，具体参数有啥用我也不是太清楚，但是调节其中一项非常有用，那就是```Device Accel Velocity Scaling```，对于我来说，设置为1感觉效果非常好，这个不同的鼠标可能不一样，建议多尝试几次。

如果效果不太明显的话可以试着调整一下```Device Accel Constant Deceleration```和```Device Accel Adaptive Deceleration```这个2个值

>这几个参数的区别可以参考这篇英文[文章](http://510x.se/notes/posts/Changing_mouse_acceleration_in_Debian_and_Linux_in_general/)，感兴趣的可以细看一下。

使用```--set-prop```就可以设置这些参数，而且是立即生效，但是重启后就会失效，可以写个脚本开机自动运行一下就可以了。
```
xinput --set-prop 10 "Device Accel Velocity Scaling" 1
```

## 2.滚轮滚动太慢
至少在chrome里面感觉很慢，遇到很长的网页滚的手疼，系统设置也没有这个选项，最早我是通过安装一个浏览器扩展插件解决这个问题的，但是后面又发现一个更简单的方法。

安装```imwheel```

然后在用户目录下创建一个```vim .imwheelrc```配置文件，写入一下配置：
```
".*"
None,      Up,   Button4, 4
None,      Down, Button5, 4
Control_L, Up,   Control_L|Button4
Control_L, Down, Control_L|Button5
Shift_L,   Up,   Shift_L|Button4
Shift_L,   Down, Shift_L|Button5
```
最主要的就是前面2个行，后面几行可以不用管，其中“4”设置的就是滚动速度，大家可以根据自己的需要设置合适的值，保存之后可以通过```killall imwheel && imwheel```重新加载配置。

简单易用，推荐使用这个方法，有些文章说还可以通过xorg设置，但是看起来比较复杂，我没试过。