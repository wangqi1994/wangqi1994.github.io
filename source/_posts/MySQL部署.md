---
title: "MySQL部署"
catalog: true
toc_nav_num: true
date: 2020-04-26 12:00:00
subtitle: "MySQL部署备忘流程"
article_type: 0 # 0 原创，1 转载
tags:
- [ruozedata]
- [Linux]
- [MySQL]
categories:
- [ruozedata]
- [Linux]
- [MySQL]
updateDate: 2020-04-26 12:00:00
top: 0  #置顶
---

### 环境：
OS：CentOS7.2 （虚拟机）
MySQL安装包版本mysql-5.7.11-linux-glibc2.5-x86_64.tar.gz

### 1.安装准备
windows下载安装包，通过rz上传到Linux系统中
[下载mysql安装包地址](https://dev.mysql.com/downloads/mysql/)

### 2.解压及创建目录
```bash
[root@hadoop001 ~]# mv mysql-5.7.11-linux-glibc2.5-x86_64.tar.gz /usr/local #移动
[root@hadoop001 local]# tar -xzvf mysql-5.7.11-linux-glibc2.5-x86_64.tar.gz#解压
[root@hadoop001 local]# mv mysql-5.7.11-linux-glibc2.5-x86_64 mysql #改名字
[root@hadoop001 local]# mkdir mysql/arch mysql/data mysql/tmp #创建目录
```

### 3.创建my.cnf文件
```bash
[root@hadoop001 local]# vi /etc/my.cnf

[client]
port            = 3306
socket          = /usr/local/mysql/data/mysql.sock
default-character-set=utf8mb4

[mysqld]
port            = 3306
socket          = /usr/local/mysql/data/mysql.sock

skip-slave-start

skip-external-locking
key_buffer_size = 256M
sort_buffer_size = 2M
read_buffer_size = 2M
read_rnd_buffer_size = 4M
query_cache_size= 32M
max_allowed_packet = 16M
myisam_sort_buffer_size=128M
tmp_table_size=32M

table_open_cache = 512
thread_cache_size = 8
wait_timeout = 86400
interactive_timeout = 86400
max_connections = 600

# Try number of CPU's*2 for thread_concurrency
# thread_concurrency = 32 

# isolation level and default engine 
default-storage-engine = INNODB
transaction-isolation = READ-COMMITTED

server-id  = 1739
basedir     = /usr/local/mysql
datadir     = /usr/local/mysql/data
pid-file     = /usr/local/mysql/data/hostname.pid

#open performance schema
log-warnings
sysdate-is-now

binlog_format = ROW
log_bin_trust_function_creators=1
log-error  = /usr/local/mysql/data/hostname.err
log-bin = /usr/local/mysql/arch/mysql-bin
expire_logs_days = 7

innodb_write_io_threads=16

relay-log  = /usr/local/mysql/relay_log/relay-log
relay-log-index = /usr/local/mysql/relay_log/relay-log.index
relay_log_info_file= /usr/local/mysql/relay_log/relay-log.info

log_slave_updates=1
gtid_mode=OFF
enforce_gtid_consistency=OFF

# slave
slave-parallel-type=LOGICAL_CLOCK
slave-parallel-workers=4
master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log_recovery=ON

#other logs
#general_log =1
#general_log_file  = /usr/local/mysql/data/general_log.err
#slow_query_log=1
#slow_query_log_file=/usr/local/mysql/data/slow_log.err

#for replication slave
sync_binlog = 500


#for innodb options 
innodb_data_home_dir = /usr/local/mysql/data/
innodb_data_file_path = ibdata1:1G;ibdata2:1G:autoextend

innodb_log_group_home_dir = /usr/local/mysql/arch
innodb_log_files_in_group = 4
innodb_log_file_size = 1G
innodb_log_buffer_size = 200M

#根据生产需要，调整pool size 
innodb_buffer_pool_size = 2G
#innodb_additional_mem_pool_size = 50M #deprecated in 5.6
tmpdir = /usr/local/mysql/tmp

innodb_lock_wait_timeout = 1000
#innodb_thread_concurrency = 0
innodb_flush_log_at_trx_commit = 2

innodb_locks_unsafe_for_binlog=1

#innodb io features: add for mysql5.5.8
performance_schema
innodb_read_io_threads=4
innodb-write-io-threads=4
innodb-io-capacity=200
#purge threads change default(0) to 1 for purge
innodb_purge_threads=1
innodb_use_native_aio=on

#case-sensitive file names and separate tablespace
innodb_file_per_table = 1
lower_case_table_names=1

[mysqldump]
quick
max_allowed_packet = 128M

[mysql]
no-auto-rehash
default-character-set=utf8mb4

[mysqlhotcopy]
interactive-timeout

[myisamchk]
key_buffer_size = 256M
sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M
```
根据生产需要，调整pool size
生产一般8G 结合实际情况设置

### 4.创建用户组及用户
```bash
[root@hadoop001 local]# groupadd -g 101 dba
[root@hadoop001 local]# useradd -u 515 -g dba -G root -d /usr/local/mysql mysqladmin
#-u uid -g 主组 -G 次组 -d 家目录设置 
[mysqladmin@hadoop001 ~]$ id
uid=515(mysqladmin) gid=101(dba) groups=101(dba),0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

## 一般不需要设置mysqladmin的密码，直接从root或者LDAP用户sudo切换
#[root@hadoop001 local]# passwd mysqladmin
Changing password for user mysqladmin.
New UNIX password: 
BAD PASSWORD: it is too simplistic/systematic
Retype new UNIX password: 
passwd: all authentication tokens updated successfully.
## if user mysqladmin is existing,please execute the following command of usermod.
#[root@hadoop001 local]# usermod -u 514 -g dba -G root -d /usr/local/mysql mysqladmin
```
### 5.copy 环境变量配置文件至mysqladmin用户的home目录中,为了以下步骤配置个人环境变量
```bash
[root@hadoop001 local]# cp /etc/skel/.* /usr/local/mysql  ###important
```
### 6.配置环境变量
```
[root@hadoop001 local]# vi mysql/.bash_profile
# .bash_profile
# Get the aliases and functions

if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs
export MYSQL_BASE=/usr/local/mysql
export PATH=${MYSQL_BASE}/bin:$PATH


unset USERNAME

#stty erase ^H
set umask to 022
umask 022
PS1=`uname -n`":"'$USER'":"'$PWD'":>"; export PS1

## end
```
建议修改.bashrc文件,如上内容不一致。

PS1后边是设置中括号内包含路径，可以省略pwd操作

### 7.赋权限和用户组，切换用户mysqladmin，安装
```bash
[root@hadoop001 local]# chown  mysqladmin:dba /etc/my.cnf 
[root@hadoop001 local]# chmod  640 /etc/my.cnf  


[root@hadoop001 local]# chown -R mysqladmin:dba /usr/local/mysql
[root@hadoop001 local]# chmod -R 755 /usr/local/mysql 
```
### 8.配置服务及开机自启动
```bash
[root@hadoop001 local]# cd /usr/local/mysql
#将服务文件拷贝到init.d下，并重命名为mysql
[root@hadoop001 mysql]# cp support-files/mysql.server /etc/rc.d/init.d/mysql 
#赋予可执行权限
[root@hadoop001 mysql]# chmod +x /etc/rc.d/init.d/mysql
#删除服务
[root@hadoop001 mysql]# chkconfig --del mysql
#添加服务
[root@hadoop001 mysql]# chkconfig --add mysql
[root@hadoop001 mysql]# chkconfig --level 345 mysql on
[root@hadoop001 mysql]# vi /etc/rc.local
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.

touch /var/lock/subsys/local

su - mysqladmin -c "/etc/init.d/mysql start --federated"

```
### 9.安装libaio及安装mysql的初始db
```bash
[root@hadoop001 mysql]# yum -y install libaio
Loaded plugins: fastestmirror, langpacks
base                                                                                                 | 3.6 kB  00:00:00     
extras                                                                                               | 2.9 kB  00:00:00     
updates                                                                                              | 2.9 kB  00:00:00     
Loading mirror speeds from cached hostfile
 * base: mirrors.cn99.com
 * extras: mirrors.163.com
 * updates: mirrors.cn99.com
Package libaio-0.3.109-13.el7.x86_64 already installed and latest version
[root@hadoop001 mysql]# sudo su - mysqladmin

hadoop001:mysqladmin:/usr/local/mysql:>bin/mysqld \
--defaults-file=/etc/my.cnf \
--user=mysqladmin \
--basedir=/usr/local/mysql/ \
--datadir=/usr/local/mysql/data/ \
--initialize

#在初始化时如果加上 –initial-insecure，则会创建空密码的 root@localhost 账号，否则会创建带密码的 root@localhost 账号，密码直接写在 log-error 日志文件中
#（在5.6版本中是放在 ~/.mysql_secret 文件里，更加隐蔽，不熟悉的话可能会无所适从）
```

### 10.查看临时密码
```bash
hadoop001:mysqladmin:/usr/local/mysql/data:>cat hostname.err |grep password
2020-04-27T05:06:23.348090Z 1 [Note] A temporary password is generated for root@localhost: VqS1#h8HgWxl
```
### 11.启动
```bash
/usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf &
```
### 12.登录及修改用户密码
```bash
hadoop001:mysqladmin:/usr/local/mysql/data:>mysql -uroot -p # 尽量不要将密码写在命令行中
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.11-log

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> alter user root@localhost identified by 'hadoop001' # 记住以分号;结束命令，修改密码
    -> ;
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'hadoop001' ;# 一般是公司DBA进行操作
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges; # 刷新权限
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye
```
### 13.重启
```bash
hadoop001:mysqladmin:/usr/local/mysql:>service mysql restart
Shutting down MySQL..2020-04-27T05:16:16.947024Z mysqld_safe mysqld from pid file /usr/local/mysql/data/hostname.pid ended
 SUCCESS! 
Starting MySQL.. SUCCESS! 
[1]+  Done                    /usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf  (wd: ~/bin)
(wd now: ~)
hadoop001:mysqladmin:/usr/local/mysql:>mysql -uroot -p
Enter password:
```
标准登录写法
mysql -u用户 -p密码 -h 机器 -P端口号 指定db（默认不加）