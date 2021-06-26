---
title: Ubuntu休眠后屏幕缩放恢复
date: 2021-06-20 11:33:10
category: Linux
tags: 
    - Ubuntu
---
现在很多人都使用高分屏，比如2k、4k的显示器，在Ubuntu上面一般都需要设置缩放比率，之前我写过一篇文章介绍过 [Ubuntu 4K显示器缩放设置](https://wangbjun.site/2019/linux/ubuntu-4k-scale.html)

但是我最近发现存在一个问题，当你电脑休眠之后再恢复，这个屏幕缩放比例就还原了，变成1了，比如我的4k显示器一般设置是1.88刚好合适，瞬间变回1那个界面UI真的酸爽。。。

多次实验发现，这个不是休眠的问题，实际上只要你用显示器上面的按钮关闭显示器或者拔掉显示器电源再插上，这个缩放比例都会被重置。

关于这个问题的解决方案我找了很久，用中文搜索一直没找到，最后用英文搜索在```ask ubuntu```上面发现有人提过这个问题：[17.04 Display scaling reverting to 1 after resume from suspend?
](https://askubuntu.com/questions/909235/17-04-display-scaling-reverting-to-1-after-resume-from-suspend)

<!--more-->

基本上可以断定这是一个Ubuntu的bug，有些人说这可能是unity的bug，但是我看回答的描述这个bug一直都没有修复，虽然我用的是16.04版本，但是后面的18.04、20.04版本都是gnome桌面了，也会出现这个问题。

最终行之有效的解决方案就在上面的问题回答中，这里面也说一下，解决方案就是写了个perl脚本，在循环里面监听系统缩放比例的变化，一旦发现有变动就会重新设置缩放比例，实测有效。

脚本代码如下：
```
#!/usr/bin/perl -w
use strict;

my $dconf_line = `dconf read /com/ubuntu/user-interface/scale-factor`;
my ($scale_factor) = $dconf_line =~ m/DP1\': (\d+)/;

if ($scale_factor) {
    print STDOUT "Current value of scale_factor: $scale_factor ...\n\n";
} else {
    die "Error: cannot find scale_factor value in dconf\n(value of /com/ubuntu/user-interface/scale-factor was $dconf_line\n\n";
}

open(my $fh, "-|", "dconf watch /com/ubuntu/user-interface/scale-factor");

while (<$fh>) {
    if (m/DP1\': (?!$scale_factor)/) {
        `dconf write /com/ubuntu/user-interface/scale-factor "{'DP1': $scale_factor}"`;
        my $date = `date`;
        print STDOUT "$date -- scaling factor adjusted\n\n";
    }
}
```
需要注意的是上面那个```DP1```是显示接口名称，不同电脑可能不一样，有些可能是DP-1，建议先在命令行里面执行命令获取一下，然后手动修改一下：
```
jwang@jun:~$ dconf read /com/ubuntu/user-interface/scale-factor
{'DP1': 15}
```

Ubuntu系统自带perl脚本运行环境，直接把这段代码保存为文件，chmod+x 添加执行权限，然后设置成开机启动就行。

开机启动可以把命令写到```/etc/rc.local```里面，或者我觉得最简单的方式就是在Ubuntu里面有一个名称叫作```Startup Applications```的应用，中文应该是叫开机启动，在这里添加一个就行。

<img src = "/images/2021/2021-06-20_12-00.png" />



