---
title: Ubuntu命令行下打造一个音乐闹钟
date: 2015-03-07 19:01:09
category: Linux
tags: 
    - Ubuntu
---

### 一.命令行播放音乐
第一次听说Linux命令行能播放歌曲我是怀疑的...一直觉得命令行干这个事情应该非常复杂，其实想想图形界面本质上只是一种交互方式，可能大家平时用的音乐播放器都有一个非常漂亮的界面,点一下就能播放音乐，但是其本质上还是调用系统API操控音响或者耳机等设备来发出声音！

言归正传，在Linux命令行下播放音乐只需要一行命令搞定：

<!--more-->

```
jwang@jwang:~$ sudo apt-get install sox libsox-fmt-all
```
然后播放歌曲只需要在其目录下面play就行：
```
jwang@jwang:~/Music/CloudMusic$ play *.mp3
jwang@jwang:~/Music/CloudMusic$ play 平凡之路.mp3
```
*.mp3是播放所有mp3歌曲，也可以指定歌曲名，Ctrl+c可以切换歌曲，即中断当前播放歌曲，切换到下一曲，这个命令是很强大，有很多可选参数，大家可以man一下

### 二.随机播放歌曲
默认情况下，播放是按照你文件中歌曲的排序顺序播放的，如何实现随机播放呢？我想了一个小技巧，写了一个shell脚本：
```
#!/bin/bash
#歌曲存放路径
dir='/home/jwang/Music/CloudMusic'

#歌曲名称列表,中间不要有空格
sounds=(
CanoninD.mp3
泡沫.mp3
演员.mp3
南山南.mp3
Beautiful.mp3
Victory.mp3
DreamItPossible.mp3
)

#产生随机数
function rand(){
    min=$1
    max=$(($2-$min+1))
    num=$(date +%s%N)
    return $(($num%$max+$min))
}
rand 0 ${#sounds[@]}-1
#执行播放命令
/usr/bin/play ${dir}/${sounds[$?]}
```
当然这也是伪随机，而且需要把歌曲名称存在数组里面，好处就在可以自定义需要播放的歌曲，坏处就说如果需要播放的歌曲很多，那就麻烦了，可以给这个脚本起一个名字比如说music，以后直接敲music就可以随机播放一首歌曲，也可以把这个命令放到环境变量里面去
```
jwang@jwang:~$ sudo ln -s /home/jwang/Documents/play.sh /usr/bin/music
jwang@jwang:~$ music
```
### 三.定时音乐闹钟
Linux下定时任务很容易配置，这里不多说，给一个例子：
```
jwang@jwang:~$ crontab -e
```
加入下面语句，意思是每天早上7点50随机播放一首歌曲，当然前提是你电脑要开机...
```
50 7 * * * nohup /usr/bin/music > /dev/null 2>&1 &
```

其实我觉得可以加一条定时任务，每隔1个小时播放一首歌曲缓解一下工作压力
```
* */1 * * * nohup /usr/bin/music > /dev/null 2>&1 &
```
好了，就说这么多了，虽然闹钟手机也能设置，但是折腾电脑玩的就是自己动手的乐趣!祝大家玩机愉快！

有一点坑的地方需要说明，就是Ubuntu下使用这个play命令播放音乐可能会产生冲突，意思就是如果你正在播放视频或者音乐，这时候的定时任务里面play命令可能无法播放成功，具体原因暂时还没找到
