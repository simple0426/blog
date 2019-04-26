---
title: 文件同步rsync
date: 2018-02-26 16:09:38
tags: ['rsync']
categories: ['linux']
---
# name
* 远程同步：remote synchronization
* 是一个快速的、全能的远程或本地的文件复制工具
* 只支持本地、本地与远程直接的文件拷贝，不支持俩个远程主机之间的文件拷贝
* 当只有源没有目标指定时，命令类似于'ls -l'

# synopsis【概要】
## 本地模式
rsync [OPTION...] SRC... [DEST]
## 通过远程shell
* Pull: rsync [OPTION...] [USER@]HOST:SRC... [DEST]
* Push: rsync [OPTION...] SRC... [USER@]HOST:DEST

## 通过rsync的daemon
* Pull
    - rsync [OPTION...] [USER@]HOST::SRC... [DEST]
    - rsync [OPTION...] rsync://[USER@]HOST[:PORT]/SRC... [DEST]
* Push
    - rsync [OPTION...] SRC... [USER@]HOST::DEST
    - rsync [OPTION...] SRC... rsync://[USER@]HOST[:PORT]/DEST

# features【特性】
* 支持拷贝链接、设备、属主、属组、权限
* 类似于tar命令一样的排除选项
* 类似于版本控制系统一样的排除模式
* 可以使用任何的远程管道方式传输，包括ssh、rsh等
* 不需要超级用户权限
* 最小花费的管道传输
* 支持匿名和认证的rsync进程模式

# options【命令选项】
>目录末尾加斜线，则只拷贝目录内的内容；与此相反，没有斜线时，会拷贝目录本身以及目录下的内容。

* -e 指定远程shell选项，如：-e 'ssh -p 50121'
* --password-file 指定连接rsync daemon的密码文件
* -C --cvs-exclude 和版本控制系统一样忽略文件
* -v ,--verbose  显示传输详情
* -z , - -compress 压缩传输
* -a ,- -archive 归档模式，表示以递归方式传输文件，并保持文件属性不变
* -P ,- -progress 显示传输进度
* --delete 保持源和目标文件的一致性，如果目标的文件在源没有则删除
* --delete-before 在传输前删除相关文件
* include与exclude：
    - --exclude=PATTERN       exclude files matching PATTERN
        + 排除部分不需要同步的目录：rsync -azvP --exclude "prod/" files/ test 
    - --exclude-from=FILE     read exclude patterns from FILE
    - --include=PATTERN       don't exclude files matching PATTERN
        +  只同步部分目录：rsync -azvP --include "pre/" --include "keyfile/" --exclude "/*" files/ test     
    - --include-from=FILE     read include patterns from FILE

# daemon模式
## 配置文件
* rsyncd.conf

```
# rsyncd.conf
uid = rsync
gid = rsync
use chroot = no
max connections = 2000
timeout = 600
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
log file = /var/log/rsyncd.log
ignore errors
read only = false
list = false
hosts allow = 172.16.1.0/24
hosts deny = 0.0.0.0/32
auth users = tridge, susan
secrets file = /etc/rsync.password
[backup]
comment = backup
path = /backup
[share]
comment = share
path = /share 
```

*  rsync.password

```
tridge:mypass
susan:herpass
```

## 服务启动
* 正常启动 
    -  默认使用873端口
    -  默认配置文件/etc/rsyncd.conf
* 自定义启动
    - --config=config_file

## daemon错误集锦
* 服务端有防火墙阻挡

```
[root@client-2 log]# rsync -avzP messages_2014-03-30.tar.gz susan@192.168.100.1::oldboy --password-file=/etc/rsyncd.secrets
rsync: failed to connect to 192.168.100.1: No route to host (113)
rsync error: error in socket IO (code 10) at clientserver.c(124) [sender=3.0.6]
```

* 服务端log显示secrets file权限有问题，此文件不能被其他人读取

```
# client
[root@client-3 mnt]# rsync -avzP /mnt rsync1@192.168.100.2::oldboy
Password: 
@ERROR: auth failed on module oldboy
rsync error: error starting client-server protocol (code 5) at main.c(1503) [sender=3.0.6]

# server
2014/04/11 16:22:26 [1768] secrets file must not be other-accessible (see strict modes option) 
```

* 客户端提示secrets file权限有问题，此文件不能被其他人读取

```
[root@client-3 mnt]# rsync -avzP /mnt rsync1@192.168.100.2::oldboy --password-file=/etc/rsyncd.secrets 
password file must not be other-accessible
continuing without password file 
```

* ssh使用系统账号登陆

```
[root@client-2 oldboy]# rsync -avzP -e "ssh -p 22" /etc/hosts susan@192.168.100.1::oldboy --password-file=/etc/rsyncd.secrets
susan@192.168.100.1's password: 
rsync error: received SIGINT, SIGTERM, or SIGHUP (code 20) at rsync.c(544) [sender=3.0.6]
```
