---
title: 一条Linux命令
date: 2018-06-08 15:01:06
category: Linux
tags: Linux
---

咱今天先从一个命令讲起，先看一个命令：

```shell
ps -ef|grep nginx|awk '{print $2}'|xargs sudo kill -9
```
上面这条命令使用了管道组合了多个命令，作用是找个所有进程名字包含 **nginx** 的进程，然后 kill 这些进程。

---

### 1.首先是ps这个命令，简单的说是查看当前系统进程。
```shell
Usage: ps [OPTION]
```
在Linux的世界里，一个看似简单的命令，其背后的参数十分丰富，功能十分强大，如果你 man 一下这个命令，其手册打印出来估计有几十页，参数多达几十个，估计能记住的人不多，
但是好在平时我们只用到其中几个参数就够用了。所以这里我也只是简单说下常用参数和常见应用场景，详细命令可以man或者help。

对于ps这个命令，按照手册的说法，它有不同的风格，有适合UNIX，有适合BSD，一般来说，**ps -axu** 和 **ps -ef** 效果是一样的。
其结果如下：
```bash
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 10:12 ?        00:00:02 /sbin/init splash
root         2     0  0 10:12 ?        00:00:00 [kthreadd]
root         3     2  0 10:12 ?        00:00:00 [rcu_gp]
root         4     2  0 10:12 ?        00:00:00 [rcu_par_gp]
root         6     2  0 10:12 ?        00:00:00 [kworker/0:0H]
root         8     2  0 10:12 ?        00:00:00 [mm_percpu_wq]
root         9     2  0 10:12 ?        00:00:00 [ksoftirqd/0]
root        10     2  0 10:12 ?        00:00:20 [rcu_sched]
root        11     2  0 10:12 ?        00:00:00 [rcu_bh]
root        12     2  0 10:12 ?        00:00:00 [migration/0]
root        13     2  0 10:12 ?        00:00:00 [idle_inject/0]
root        15     2  0 10:12 ?        00:00:00 [cpuhp/0]
root        16     2  0 10:12 ?        00:00:00 [cpuhp/1]
root        17     2  0 10:12 ?        00:00:00 [idle_inject/1]
root        18     2  0 10:12 ?        00:00:00 [migration/1]
root        19     2  0 10:12 ?        00:00:00 [ksoftirqd/1]
root        21     2  0 10:12 ?        00:00:00 [kworker/1:0H-kb]
root        22     2  0 10:12 ?        00:00:00 [cpuhp/2]
root        23     2  0 10:12 ?        00:00:00 [idle_inject/2]
.......
.......
.......
```
---

### 2.grep命令
```shell
Usage: grep [OPTION]... PATTERN [FILE]...
```
这个命令是用来搜索文本内容，支持丰富的参数, 最简单的用法：
```bash
jwang@jwang:~$ grep server /etc/nginx/nginx.conf 
	# server_tokens off;
	# server_names_hash_bucket_size 64;
	# server_name_in_redirect off;
	ssl_prefer_server_ciphers on;
#	server {
#	server {
jwang@jwang:~$ 

```
上面的命令是打印出nginx.conf文件里面所有包含server文字的行，默认情况下，这个搜索是模糊匹配，而且是区分大小写的。

常用参数：
```shell
-i ：不忽略大小写 
-n ：显示行号 
-c ：显示匹配的数量 
-v ：反向选择，亦即显示出没有 ‘搜寻字符串’ 内容的那一行
-r ：递归搜索目录下所有文件
```
这个还支持正则表达式搜索，我平时用的少，大部分时候普通字符串就够用了。

还有几个挺有意思的参数：

```shell
  -B, --before-context=NUM  打印出搜索结果的前NUM行
  -A, --after-context=NUM   打印出搜索结果的后NUM行
  -C, --context=NUM         打印出搜索结果的前后NUM行
```
举个例子 ```grep -C 2 jwang /etc/passwd```

```bash
jwang@jwang:~$ grep -C 2 jwang /etc/passwd
saned:x:119:127::/var/lib/saned:/bin/false
usbmux:x:120:46:usbmux daemon,,,:/var/lib/usbmux:/bin/false
jwang:x:1000:1000:JWang,,,:/home/jwang:/bin/bash
nvidia-persistenced:x:121:129:NVIDIA Persistence Daemon,,,:/:/sbin/nologin
mysql:x:122:131:MySQL Server,,,:/nonexistent:/bin/false
```
所以 ```ps -ef|grep nginx```的结果如下：
```bash
root     26642     1  0 17:36 ?        00:00:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data 26643 26642  0 17:36 ?        00:00:00 nginx: worker process
www-data 26644 26642  0 17:36 ?        00:00:00 nginx: worker process
www-data 26645 26642  0 17:36 ?        00:00:00 nginx: worker process
www-data 26646 26642  0 17:36 ?        00:00:00 nginx: worker process
www-data 26647 26642  0 17:36 ?        00:00:00 nginx: worker process
www-data 26648 26642  0 17:36 ?        00:00:00 nginx: worker process
www-data 26649 26642  0 17:36 ?        00:00:00 nginx: worker process
www-data 26650 26642  0 17:36 ?        00:00:00 nginx: worker process
jwang    28782 18097  0 18:31 pts/20   00:00:00 grep --color=auto nginx
```

---

### 3.awk命令
>Awk是一种便于使用且表达能力强的程序设计语言，可应用于各种计算和数据处理任务。

看这介绍就知道awk多强大，都上升到语言的层次，先说说一开始的命令里面用法: ```awk '{print $2}'```

默认情况下，awk使用 **空格** 去分割字符串，把上面的结果每一行按照空格去分割成N块，其中$0代表字符串本身，$1代表第一个块，$2代表第二个块，以此类推....

所以```ps -ef|grep nginx|awk '{print $2}'```的结果是：
```bash
26642
26643
26644
26645
26646
26647
26648
26649
26650
28836
```
awk常用参数：
```shell
1. -F fs or --field-separator fs 指定输入文件折分隔符，fs是一个字符串或者是一个正则表达式
2. -v var=value or --asign var=value 赋值一个用户定义变量。
3. -f scripfile or --file scriptfile 从脚本文件中读取awk命令。
```

关于awk脚本，我们需要注意两个关键词BEGIN和END。

BEGIN{ 这里面放的是执行前的语句 }

END {这里面放的是处理完所有的行后要执行的语句 }

{这里面放的是处理每一行时要执行的语句}

>假设有这么一个文件（学生成绩表）：
```
$ cat score.txt
Marry   2143 78 84 77
Jack    2321 66 78 45
Tom     2122 48 77 71
Mike    2537 87 97 95
Bob     2415 40 57 62
```
我们的awk脚本如下：
```bash
$ cat cal.awk
#!/bin/awk -f
#运行前
BEGIN {
    math = 0
    english = 0
    computer = 0
 
    printf "NAME    NO.   MATH  ENGLISH  COMPUTER   TOTAL\n"
    printf "---------------------------------------------\n"
}
#运行中
{
    math+=$3
    english+=$4
    computer+=$5
    printf "%-6s %-6s %4d %8d %8d %8d\n", $1, $2, $3,$4,$5, $3+$4+$5
}
#运行后
END {
    printf "---------------------------------------------\n"
    printf "  TOTAL:%10d %8d %8d \n", math, english, computer
    printf "AVERAGE:%10.2f %8.2f %8.2f\n", math/NR, english/NR, computer/NR
}
```
我们来看一下执行结果：
```
$ awk -f cal.awk score.txt
NAME    NO.   MATH  ENGLISH  COMPUTER   TOTAL
---------------------------------------------
Marry  2143     78       84       77      239
Jack   2321     66       78       45      189
Tom    2122     48       77       71      196
Mike   2537     87       97       95      279
Bob    2415     40       57       62      159
---------------------------------------------
  TOTAL:       319      393      350
AVERAGE:     63.80    78.60    70.00
```

>再看一个案例，查出nginx日志里面状态为500的请求:

```
awk '$9 == 500 {print $0}' /var/log/nginx/access.log
```

awk还支持常见的if while等逻辑控制语句。

---

### 4.xargs命令
```bash
Usage: xargs [OPTION]... COMMAND [INITIAL-ARGS]...
```
这个命令作用是使用接收的内容当作参数去执行一条命令，一般都是配合管道使用，比如说在上面的例子里面，xargs的作用就是接收前面的pid，然后执行kill命令。

再看个例子： ```sudo find / -name nginx |xargs ls -l```

这个命令意思是列出所有目录名或者文件名包含nginx的详情，其结果大概是这样：
```bash
-rw-r--r-- 1 root  root      389 2月  12  2017 /etc/default/nginx
-rwxr-xr-x 1 root  root     4579 2月  12  2017 /etc/init.d/nginx
-rw-r--r-- 1 root  root      329 2月  12  2017 /etc/logrotate.d/nginx
-rw-r--r-- 1 root  root      374 2月  12  2017 /etc/ufw/applications.d/nginx
-rwxr-xr-x 1 root  root  1230768 7月  12  2017 /usr/sbin/nginx

/etc/nginx:
total 56
drwxr-xr-x 2 root root 4096 7月  12  2017 conf.d
-rw-r--r-- 1 root root 1077 2月  12  2017 fastcgi.conf
-rw-r--r-- 1 root root 1007 2月  12  2017 fastcgi_params
-rw-r--r-- 1 root root 2837 2月  12  2017 koi-utf
-rw-r--r-- 1 root root 2223 2月  12  2017 koi-win
-rw-r--r-- 1 root root 3957 2月  12  2017 mime.types
-rw-r--r-- 1 root root 1462 2月  12  2017 nginx.conf
-rw-r--r-- 1 root root  180 2月  12  2017 proxy_params
-rw-r--r-- 1 root root  636 2月  12  2017 scgi_params
drwxr-xr-x 2 root root 4096 11月  7 14:01 sites-available
drwxr-xr-x 2 root root 4096 11月  7 14:01 sites-enabled
drwxr-xr-x 2 root root 4096 5月   6  2018 snippets
-rw-r--r-- 1 root root  664 2月  12  2017 uwsgi_params
-rw-r--r-- 1 root root 3071 2月  12  2017 win-utf

/home/jwang/Documents/Work/trunk/webroot/static/lib/codemirror/mode/nginx:
total 20
-rw-rw-r-- 1 jwang jwang  5230 5月   7  2018 index.html
-rw-rw-r-- 1 jwang jwang 10169 5月   7  2018 nginx.js

/usr/share/doc/nginx:
total 12
lrwxrwxrwx 1 root root   33 7月  12  2017 changelog.Debian.gz -> ../nginx-core/changelog.Debian.gz
-rw-r--r-- 1 root root 8641 2月  12  2017 copyright

/usr/share/nginx:
total 4
drwxr-xr-x 2 root root 4096 5月   6  2018 html
```
所以，最后的xargs命令是把前面筛选得到的pid作为参数传给命令kill执行，有时候会有权限问题，所以这里加了个sudo。


