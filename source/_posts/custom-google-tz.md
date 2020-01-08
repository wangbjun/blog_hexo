---
title: 自建梯子访问Google等学术网站
date: 2018-12-09 08:01:02
category: Linux
tags: 
    - Google
---

这个话题有点敏感，但是我首先说明一下，此处只做一个记录，网上类似的文章超级多，而且我纯粹是拿来上Google学习用，搞web开发的应该没几个不用Chrome的，Chrome配上Google账号同步简直完美。

<!--more-->

### 1.首先，你得有一个国外的服务器
购买国外服务器的途径很多，比如阿里云就有很多国外的主机，缺点是较贵，并且感觉不安全，你们懂的。还有比如说亚马逊云等云服务器商也可以买到国外主机。还有一些国外vps主机也可以。

国外的主机相对来说便宜点，比如vultr的vps最便宜的3.5美元一个月，单核，512M，500G流量，搭梯子绰绰有余，除非你一天到晚看YouTube，不然正常浏览网页，我一个月10G都用不完。国外主机的缺点就是英文，而且很多要求使用信用卡支付，比较麻烦。（现在vultr已经支持支付宝和微信支付）

至于各个服务器哪个好哪个坏我就不评论了，用的不多，但是我可以给大家一个选择标准:

##### 1.地理位置

如果你只是自己用，你肯定得选个离自己近的主机，比如常见的日本的适合北方，香港的适合南方，当然美国的就稍微远点

##### 2.ping值

拿到主机先ping一下，有很多vps的ip已经被ban掉了，还有的丢包严重，或者延迟特别大，比如说美国洛杉矶的主机一般延迟在200ms左右，纽约稍微高点，日本的大概100ms左右，其实吧，这些影响都不大，毕竟你是拿来上网页，又不是打游戏，对ping要求不高。

然后你可以试着在vps上面放一个文件，下载试试，测试一下带宽

一般来说，由于中国的地域广阔，各个地方的宽带都不一样，有电信网通，还有乱七八糟的小宽带，适合别人的不一定适合你的，所以适合自己的最好！

### 2.安装shadowsocks
首先，大家都这个软件得有一个概念，这个软件是一款支持端对端加密的代理软件，所以他有2个端，一个是服务器端，另一个是客户端，这里只说服务器端。

```
apt-get install python-pip python-setuptools
pip install git+https://github.com/shadowsocks/shadowsocks.git@master
```
这2条命令就可以搞定安装，https://github.com/shadowsocks/shadowsocks.git 是这个项目的github地址，有兴趣的可以去看看，上面也有安装文档。

### 3.配置shadowsocks
如果上面的安装没有报错，那么在安装完成后，应该会有2个命令可以用，一个是sslocal，一个是ssserver。sslocal其实是客户端用的，但是现在很多客户端都是GUI的，github上面有各个平台的客户端可以下载安装使用。

ssserver 才是服务器端会用到的, 有兴趣的可以help一下，这里直接说一个最简单用法：

```
ssserver -p 443 -k password -m aes-256-cfb
```
上面命令的意思是在服务器的443端口启动一个shadowsocks，并且密码是password，加密方式是aes-256-cfb

但是实际应用里面，一般的都是多端口多密码，这样可以给很多人用，比如你的同事朋友，不过这里建议大家不要拿去售卖盈利哦，据说有被抓的，自己用用就行了！

新建一个文件 shadowsocks.json
```
{
    "server":"45.xx.12.xx",
    "local_address":"127.0.0.1",
    "local_port":1080,
    "port_password": {
        "5555": "12345678",
        "6666": "87654321",
    },
    "method": "aes-128-cfb",
    "timeout": 600,
    "fast_open":true
}
```
然后使用命令```ssserver -c shadowsocks.json```就可以启动多端口多密码配置，其中 port_password 就是端口和对应的密码，其它参数比如 method 是加密方式，这个随意，比如aes 256 理论上比128安全点，但是消耗性能，timeout是超时时间，不要太短

简单说一下客户端咋使用，一般shadowsocks的客户端都需要填服务器ip，服务器端口，服务器密码，加密方式，本地绑定ip，本地端口，其实就是这个配置文件里面的东西，前4个肯定是必须有，最后2个不一定。

### 4.服务器优化

#### 1.BBR

这个是Google的发明的tcp新的拥堵算法，对网络协议了解的人应该知道拥堵算法，简单说这个BBR对于弱网的情况下有一定的加速效果，比较适合网络查的情况下，我觉得应该有一定作用，大家可以试试！

开启BBR的需要比较新的内核，据说是必须大于4.9,uname -r 可以查看，如果不是则需要手动安装最新的内核，不过一般vps的Linux版本都比较新，反正我是没遇到这种情况，我一般用Ubuntu 16.04 或者18.04。

执行 ```lsmod | grep bbr```，如果结果中没有 tcp_bbr 的话就先执行

```
modprobe tcp_bbr
echo "tcp_bbr" | sudo tee --append /etc/modules-load.d/modules.conf
```
然后在执行:
```
echo "net.core.default_qdisc=fq" | sudo tee --append /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee --append /etc/sysctl.conf
```
保存生效:
```
sysctl -p
```

```
sysctl net.ipv4.tcp_available_congestion_control
sysctl net.ipv4.tcp_congestion_control
```
如果结果都有 bbr, 则证明你的内核已开启 bbr
```
root@vultr:~# lsmod | grep bbr
tcp_bbr                20480  31
```

#### 2.调整Linux网络配置

编辑 /etc/sysctl.conf 文件，加入以下配置：
```
fs.file-max = 51200
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.netdev_max_backlog = 250000
net.core.somaxconn = 4096
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.ip_local_port_range = 10000 65000
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_mem = 25600 51200 102400
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_mtu_probing = 1
```
保存，然后```sysctl -p```

最后再说一点，为了方便重启，大家可以搞一个开机自启脚本把启动shadowsocks的命令写里面，或者简单点直接使用 supervisor，这里贴一个supervisor的配置：
```
[program:shadowsocks]
autorestart=true
autostart=true
redirect_stderr=true
command=/usr/local/bin/ssserver -c /root/shadowsocks.json
user=root
stdout_logfile=/var/log/shadowsocks.log
```
