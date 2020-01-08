---
title: Mysql主从复制实践
date: 2018-09-05 19:10:03
category: 编程开发
tags: 
    - MySQL
    - 数据库
---

## 1.安装
很多人都知道可以用apt或者yum安装，但是实际生产环境很少采用这种方式安装，有些会采用源码编译(据说性能高？)，有些会从官网下载编译好的二进制安装包！

为什么不直接用命令安装呢？因为命令安装的位置不同发行版不一样，而且其配置文件存放的位置又各有差异，现实中大部分公司都有一个约定的规则，比如说所有的安装都安装在 **/data** 目录下，如果需要开机自启，需自行编写脚本，不依赖系统服务。

还有一个重要的原因是因为很多时候数据库是安装在单独的数据库服务器，但是一台电脑比如说32核64G内存这样的配置，是需要安装多个Mysql实例的，用不同的端口区分，这些库可能是不同的项目所用到。

<!--more-->

下面我就介绍如何使用编译好的二进制安装包安装MySQL:

1.首先下载MySQL安装包，地址：https://dev.mysql.com/downloads/mysql/ 选择符合自己需要的下载 

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1fvpadrk2mvj210s0eygnn.jpg)

这里以8.0.12版本为例，下载tar压缩包后解压，其目录结构如下：
```
drwxrwxr-x  2 jwang jwang   4096 Sep 28 15:38 bin
drwxrwxr-x  2 jwang jwang   4096 Sep 28 15:38 docs
drwxrwxr-x  3 jwang jwang   4096 Sep 28 15:37 include
drwxrwxr-x  5 jwang jwang   4096 Sep 28 15:38 lib
-rw-r--r--  1 jwang jwang 301518 Jun 29 00:18 LICENSE
drwxrwxr-x  4 jwang jwang   4096 Sep 28 15:37 man
-rw-r--r--  1 jwang jwang    687 Jun 29 00:18 README
drwxrwxr-x 28 jwang jwang   4096 Sep 28 15:38 share
drwxrwxr-x  2 jwang jwang   4096 Sep 28 15:38 support-files
```
其中bin目录存放的就是各种可执行文件

假设现在解压后的文件夹名字叫mysql8，位于 /data 目录下

先做准备一些工作, 创建mysql用户，分配权限
```
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
chown mysql:mysql mysql8
```
接下来有一个非常重要的操作，就是初始化MySQL
```
/data/mysql8/bin/mysqld --initialize --user=mysql
```
默认情况下, 上面这个操作会在一些目录创建一些文件，然而实际操作中，我们一般会指定一些配置参数,创建一个文件 /data/3306/my.cnf, 这里有一个配置文件供大家参考:
```
[mysqld]
server_id = 1100
user    = mysql
port    = 3306
datadir = /data/3306/data
basedir = /data/mysql8
log-bin = /data/3306/data/binlog
socket  = /data/3306/tmp/mysql.sock
pid-file = /data/3306/mysql.pid
log-error = /data/3306/log/mysql_error.log
relay-log = /data/3306/relaylog
relay-log-index = /data/3306/relaylog.index

default-storage-engine = Innodb
sql-mode=NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES

#慢查询
long_query_time = 1
slow-query-log = on
slow_query_log_file = /data/mysqld/3306/log/mysql_slow.log
#记录更多的日志
log_slow_admin_statements

#master/slave
slave_skip_errors=1032,1062
log-slave-updates
master-info-repository=TABLE
report_host=192.168.1.100
report_port=3306
enforce_gtid_consistency
gtid-mode=ON

#charset
character-set-server = utf8

#log_warnings = 0
open_files_limit    = 10240
#参考:短时间内最大连接
back_log = 1024

#binlog
binlog_cache_size = 4M
binlog_format = MIXED
max_binlog_cache_size = 8M
max_binlog_size = 1G
expire_logs_days = 2

#cache
query_cache_type = 1
query_cache_limit = 2M
query_cache_size = 64M

#buffer
join_buffer_size = 32M
sort_buffer_size = 32M
#???
read_rnd_buffer_size = 16M

#innodb Dynamic=NO
innodb_read_io_threads = 8
innodb_write_io_threads = 4 
innodb_buffer_pool_size = 10240M 
#数据刷新方式
innodb_flush_method = O_DIRECT
#单个连接所分配的内存大小
innodb_sort_buffer_size = 4M

#只限slave配置
innodb_flush_log_at_trx_commit = 0

#thread
thread_cache_size = 256

#connections
max_connections = 2048 
max_connect_errors = 10240
init-connect='SET NAMES utf8'
#跳过反向解析
skip-name-resolve = 1

explicit_defaults_for_timestamp = TRUE
#调用group_cat
group_concat_max_len = 204800
```
具体的配置项这么不细说了，有些可能需要根据你服务器的配置做一些调整！然后执行下面的命令初始化：
```
/data/mysql8/bin/mysqld --defaults-file=/data/3306/my.cnf --initialize --explicit_defaults_for_timestamp --user=mysql
```
如果没有报错，你应该可以在 /data/3306 目录里面看到一些生成的文件，请注意这时候在 /data/3306/log/mysql_error.log 文件里面会有一个生成的临时密码,
类似这样的语句： A temporary password is generated for root@localhost: h9iec,Z,Hel1

然后使用下面语句启动MySQL
```
/data/mysql8/bin/mysqld_safe --defaults-file=/data/3306/my.cnf --ledir=/data/mysql8/bin/ &
```
如果没有报错，你可以使用ps查看一下进程，应该是启动了！

可以使用以下命令连接MySQL：
```
/data/mysql8/bin/mysql -S /data/3306/mysql.sock -uroot -p
```
修改密码：
```
set password=password('yourpass');
flush privileges;
```
>这里需要注意的是 server_id 不能重复，建议以ip最后2位为参考，假设这里主服务器ip为192.168.1.100，从服务器ip为192.168.1.105

## 2.配置主从
看到这里说明这两台Mysql服务器已经跑起来了，接下来就是配置主从关系

首先，得在主服务器MySQL里面新建一个账号专门用于同步：
```
create user 'repl'@'192.168.%' identified by 'repl_pass';
grant select,replication slave, REPLICATION CLIENT on *.* to 'repl'@'192.168.%';
```
为了安全考虑，可以限定其ip范围，并且只授予给定权限，当然你也可以用root账号，只要有权限，应该都没问题

如果你的主服务器已经有数据的话，有2种选项，一种是不做处理，建立主从关系之后让MySQL自动同步，但是如果数据量大的话可能比较慢，另一种，
在主库上面备份数据，导入从服务器，这里有一个备份命令可以参考：
```
/data/mysql8/bin/mysqldump --skip-lock-tables --single-transaction --flush-logs --hex-blob --master-data=2 --databases yourdatabases -S/data/3306/mysql.sock -uroot -pyourpass --result-file=/data/backup.sql
```
如果你的数据真的非常大，建议在导入从库的时候在备份的文件里面加入一行配置暂时关闭binlog：
```
sed -i "1iset sql_log_bin=off;\n" /data/backup.sql
```
一切搞定之后，只剩下最后一步了，设置主从关系：
```
stop slave;
CHANGE MASTER TO MASTER_HOST='192.168.1.100', 
MASTER_PORT=3306,
MASTER_USER='repl', 
MASTER_PASSWORD='repl_pass', 
#MASTER_LOG_FILE='binlog.004335', 
#MASTER_LOG_POS=120; 
start slave;
```
注意，如果你是导入主库的数据话，你会发现在备份的文件前面有一行是这样的：
```
 CHANGE MASTER TO MASTER_LOG_FILE='binlog.004335', MASTER_LOG_POS=120;
```
这就是注释里面需要的binlog文件和其位置，全新的库的话就不需要

最后，show slave status\G 查看一下从库的状态
