---
title: Ubuntu屏幕撕裂问题
date: 2021-07-06 21:03:00
category: Linux
tags: 
    - Ubuntu
---
Linux下的显示画面撕裂（Tearing）问题由来已久，如果你观察比较仔细，肯定会发现使用Ubuntu播放视频的时候画面明显有撕裂的现象，这倒不是什么大问题，但是却一直以来存在。

因为我是使用Ubuntu当主力机，每天不仅拿来编程，也拿来娱乐，看视频电影，浏览网页，对这个现象比较注意，其实这种问题有时候也不单单是Linux存在的问题，很多玩游戏的朋友应该知道很多游戏都有一个垂直同步的选项，这也是为了解决画面撕裂问题。

这种画面撕裂问题本质上是显卡输出和显示器刷新频率不同步导致的问题，这里记录一下我的解决方式，一个是自己备用，另外分享给需要的朋友。

<!--more-->

在Ubuntu系统下，分2种情况下：

如果你是独显，比如Nvidia显卡，可以在设置面板里面勾选垂直同步选项，实际上这个效果并不明显。

后来，我换上了Intel集显，找到了一个非常有效的方式，打开 **/etc/X11/xorg.conf** 文件，这个文件是自动生成的，系统应该都有。

其实内容大概如下：
```
...
Section "Device"
        ### Available Driver options are:-
        ### Values: <i>: integer, <f>: float, <bool>: "True"/"False",
        ### <string>: "String", <freq>: "<f> Hz/kHz/MHz",
        ### <percent>: "<f>%"
        ### [arg]: arg optional
        #Option     "Accel"              	# [<bool>]
        #Option     "AccelMethod"        	# <str>
        #Option     "Backlight"          	# <str>
        #Option     "CustomEDID"         	# <str>
        #Option     "DRI"                	# <str>
        #Option     "Present"            	# [<bool>]
        #Option     "ColorKey"           	# <i>
        #Option     "VideoKey"           	# <i>
        #Option     "Tiling"             	# [<bool>]
        #Option     "LinearFramebuffer"  	# [<bool>]
        #Option     "HWRotation"         	# [<bool>]
        Option     "VSync"              	"True"
        #Option     "PageFlip"           	# [<bool>]
        #Option     "SwapbuffersWait"    	# [<bool>]
        #Option     "TripleBuffer"       	# [<bool>]
        #Option     "XvPreferOverlay"    	# [<bool>]
        #Option     "HotPlug"            	# [<bool>]
        #Option     "ReprobeOutputs"     	# [<bool>]
        #Option     "XvMC"               	# [<bool>]
        #Option     "ZaphodHeads"        	# <str>
        #Option     "VirtualHeads"       	# <i>
        Option     "TearFree"           	"True"
        #Option     "PerCrtcPixmaps"     	# [<bool>]
        #Option     "FallbackDebug"      	# [<bool>]
        #Option     "DebugFlushBatches"  	# [<bool>]
        #Option     "DebugFlushCaches"   	# [<bool>]
        #Option     "DebugWait"          	# [<bool>]
        #Option     "BufferCache"        	# [<bool>]
    Identifier  "Card0"
    Driver      "intel"
    BusID       "PCI:0:2:0"
EndSection
...
```
这里只截取部分片段，其中关键就是Option字段的配置，可以看到这里其实有很多选项，默认情况下都是注释掉的，但是有2个选项我打开了，分别是VSync、TearFree，设置为True。

然后重启就会发现画面撕裂问题解决了，至少在我的CPU 8700k的集显下，带动4k显示器没问题，非常流畅，但是为什么这个配置默认被注释掉也是很奇怪，必然有原因的，个人感觉可能影响屏幕画面响应速度，总之，在我电脑上我感觉没啥问题，如果你对画面撕裂问题不可接受，不妨设置一下试试。