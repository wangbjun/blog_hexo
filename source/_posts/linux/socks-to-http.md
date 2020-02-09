---
title: 科学上网之socks代理转http(s)
date: 2018-11-03 19:02:46
tags: Socks
category: Linux
---

从事IT开发行业的总免不了用用Google，看看国外互联网，但是呢有道墙大家都懂的，其中有一种socks5的梯子，一般都是配置浏览器，如何在命令行下也使用呢？有过使用经验的都知道，Linux终端是不走socks代理配置的，除此之外，很多软件或者应用也不支持socks代理设置，但是有一个软件是可以把socks代理转为http代理，这个软件就叫做privoxy，下面简单介绍下这个软件使用：

<!--more-->

1.开启某SS代理，代理地址假设为 127.0.0.1:1080

2.安装privoxy，这里只介绍debian系列发行版，其他系统不多说，其实都差不多
```
sudo apt-get install privoxy
```

3.修改privoxy配置
```
sudo vim /etc/privoxy/config
```

在里面添加一条：
```
forward-socks5   /               127.0.0.1:1080 .
```
请注意后面有一个 .
```
# 下面还存在以下一条配置，表示privoxy监听本机8118端口
# 把它作为http代理，代理地址为 http://localhost.8118/ 
# 可以把地址改为 0.0.0.0:8118，表示外网也可以通过本机IP作http代理
# 这样，你的外网IP为1.2.3.4，别人就可以设置 http://1.2.3.4:8118/ 为http代理

listen-address localhost:8118
```
4.重启privoxy服务
```
sudo service prioxy restart
```

5.配置命令行或者应用

如果是软件，直接在软件代理设置填写http(s)地址为 127.0.0.1:8118 即可

如果是Linux命令行可以使用export命令临时设置代理，命令如下：
```
export http_proxy=http://127.0.0.1:8118/
export https_proxy=http://127.0.0.1:8118/
```
如果需要永久设置代理，可以修改环境配置文件，把上面的命令写到.bashrc文件里面就行了，但是不建议这么做，这样所有流量都会走代理，会影响访问国内网站的速度，建议需要的时候临时配置即可，毕竟这样的场景并不多！

