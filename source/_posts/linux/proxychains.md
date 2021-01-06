---
title: Google Earth 代理配置（linux proxychains）
date: 2021-01-06 14:00:00
category: Linux
tags: 
    - Linux
    - Google
---

## 1.问题
Google Earth有多神，大家都知道，户外必备，独一无二的高清无码卫星地图深得人心。

Google Earth既有手机版也有网页版，还有Windows、Linux的桌面版本，官网都有下载。其中手机版和网页版都得FQ，桌面版一直可以直接打开（本人一直用的Linux版本），直到最近，具体哪天我忘记了，大概就在2020年12月，突然就打不开了，一直卡在主页面，我还以为自己电脑问题，重置修复+重新安装还是不行，直到看到一个新闻才知道也遭黑手，至此所有版本全军覆灭。

<!--more-->

<img src="/images/2021/2021-01-06_14-58.png" /> 

FQ倒不是什么难事，用Google服务的都有梯子，但是这个软件自身没有代理相关设置，也不走系统全局代理，可算是难倒了我。

之前也试了在命令行里面启动，通过设置环境变量的方式，结果也是不行：

```shell
export http_proxy=127.0.0.1:1081
export https_proxy=127.0.0.1:1081
```

## 2.神器ProxyChains
这个问题一直困扰着我，直到我发现了一个神器 proxychains，这玩意干啥用呢，简单的说，它可以让一些不走系统代理也不走环境变量的Linux应用走代理。

举个例子，Ubuntu下更新系统的命令 **sudo apt-get update** 默认是不走代理，虽然可以改文件配置走代理，但是还比较麻烦，但是通过proxychains就简单多了，只需在前面加一个**proxychains**即可：

```shell
jwang@jun:~$ sudo proxychains apt-get update
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-127.0.0.1:1080-|S-chain|-<>-127.0.0.1:1080-<>-127.0.0.1:1081-<>-127.0.0.1:1081-<><>-127.0.0.1:1081-|S-chain|-<>-127.0.0.1:1080-<>-127.0.0.1:1081-<><>-127.0.0.1:1081-<><>-OK
<><>-OK
<><>-127.0.0.1:1081-|S-chain|-<>-127.0.0.1:1080-<><>-OK
0% [Working]<><>-127.0.0.1:1081-<><>-OK
|S-chain|-<>-127.0.0.1:1080-<>-127.0.0.1:1081-<><>-127.0.0.1:1081-<><>-OK
Get:1 https://download.docker.com/linux/ubuntu xenial InRelease [66.2 kB]
Hit:2 https://dl.yarnpkg.com/debian stable InRelease      
Hit:3 https://linux.teamviewer.com/deb stable InRelease                              
Get:4 https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages [17.4 kB]
Hit:5 https://download.virtualbox.org/virtualbox/debian xenial InRelease            
Hit:6 http://ppa.launchpad.net/deadsnakes/ppa/ubuntu xenial InRelease
...
...
...
```
其实Linux下很多命令行应用都存在这个问题，既不走代理也不走环境变量，比如git、docker、ssh等，使用proxychains可以一网打尽，既简单又灵活。

## 3.ProxyChains配置
Ubuntu 16.04下可以通过apt直接安装，版本是3.1
```shell
jwang@jun:~$ proxychains 
ProxyChains-3.1 (http://proxychains.sf.net)
    usage:
        proxychains <prog> [args]
```
上面这个网址就是其官网，里面有一些介绍，实际上这个版本已经不维护了，github上面有一个叫作 **proxychains-ng**的项目，算是这个项目的延续项目，我对新版本新功能没有什么追求，感兴趣的可以关注一下，地址：https://github.com/rofl0r/proxychains-ng

安装完成后，Ubuntu下默认的配置文件在 **/etc/proxychains.conf**，我们需要修改最后一栏，加入自己的代理配置，比如我本机1080端口有一个sock5的代理，然后1081端口有个http代理：
```shell
# ProxyList format
#       type  host  port [user pass]
#       (values separated by 'tab' or 'blank')
#
#
#        Examples:
#
#               socks5  192.168.67.78   1080    lamer   secret
#               http    192.168.89.3    8080    justu   hidden
#               socks4  192.168.1.49    1080
#               http    192.168.39.93   8080    
#               
#
#       proxy types: http, socks4, socks5
#        ( auth types supported: "basic"-http  "user/pass"-socks )
#
[ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
socks5  127.0.0.1 1080
#http    127.0.0.1 1081
```
请注意，这块其实配置一个代理即可，因为chain本身的意思就是链条，当你配置多个代理的时候，它会像链条一样依次调用，至少在当前这种应用场景下意义并不大，毕竟背后都是一个代理服务器。

但如果你有多个代理，又想随机使用一个或者多个，这块就有点用了，它可以配置几种模式，比如dynamic_chain、strict_chain（默认）、random_chain，感兴趣的可以自己研究一下，这里不多说了。

配置文件的解释注释很全，还有一些配置参考，比如说教你有账户密码的代理怎么配，除此之外，还有一个地方要注意，就是下面的proxy_dns建议注掉，默认是打开的，我之前一直报错，不知道为什么

另外，quiet_mode建议打开，这样就不会打印多余的日志了
```shell
# Make sense only if random_chain
#chain_len = 2

# Quiet mode (no output from library)
quiet_mode

# Proxy DNS requests - no leak for DNS data
#proxy_dns 

# Some timeouts in milliseconds
tcp_read_time_out 15000
tcp_connect_time_out 8000
```

<img src="/images/2021/2021-01-06_15-51.png" /> 

但你把这些准备工作做完之后，在命令行下执行 **proxychains google-earth-pro** 完美打开，这时候所有的网络连接都会走代理，再也不卡了。

## 4.其它OS
Proxychains是支持Mac系统的，请各位自行安装测试，另外Windows的话是不支持的，但是有一个更好用的软件叫作 **Proxifier**。