---
title: 解决sudo command not found 报错
date: 2019-06-02 11:29:06
category: Linux
tags: 
    - Sudo
---

偶尔发现的一个问题，平时主要使用 **Ubuntu** 操作系统，有时候安装一些软件会用加一些自定义PATH，往往为了方便都会把配置写到  **/etc/environment** 里面,这样所有用户包括root都有效：
```
jwang@jwang:~$ cat /etc/environment 
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:
/bin:/usr/games:/usr/local/games:/home/jwang/MyBin:/opt/go/bin"
export GOPATH=/home/jwang/Go
```

<!--more-->

比如说安装了go，在使用 **sudo go** 这样命令的时候会报错，但是切换到 root 用户却没有问题，使用普通用户也没问题，查了一下发现原来 sudo 里面有一些配置：

![](https://upload-images.jianshu.io/upload_images/3571187-2c5fdea0a1fd3101.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
sudo visudo
```
![](https://upload-images.jianshu.io/upload_images/3571187-319d779b9cae7fc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个 **/etc/sudoers** 文件里面，有一个secure_path配置，大家一看就知道了，它的意思当你使用 **sudo+command** 这种形式执行命令的时候会从其配置的路径里面寻找命令，肯定是没有你自定义的PATH的，这个主要是安全考虑。

**解决方法**有几种：
1. 直接把自定义PATH路径配置在secure_path里面，简单粗暴，就是有点麻烦

2. 将 Defaults env_reset 改成 Defaults !env_reset 取消掉对PATH变量的重置，然后在.bashrc中最后添加alias sudo='sudo env PATH=$PATH'，这个感觉更麻烦

3. 直接把这3行注释掉，经测试完全没有任何问题

我是采用第3种方式解决的，非常好用，暂时未发现问题！
