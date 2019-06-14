---
title: 日志切割工具logrotate
tags: logrotate
categories: linux
date: 2019-06-13 11:13:28
---

# 示例
```
/home/haproxy/log/*.log {
rotate 15
daily
dateext
missingok
notifempty
compress
delaycompress
copytruncate
size 50M
# create 0640 user group
# olddir oldlog
# sharedscripts
# postrotate
#  reload rsyslog >/dev/null 2>&1 || true
# endscript
}
```
# [介绍][lograte]
* 功能
    - 日志轮转
    - 压缩
    - 删除
* 默认在每天的定时任务【cron.daily】下执行，因此每天只会执行一次
* 只有当日志轮转参数基于日志大小并且多次运行logrotate，或者使用-f参数【logrotate -f】才会形成对日志的修改
* 可以在命令行使用任意多的配置文件或定义配置文件目录；具有相同功能的配置选项，后面的配置会覆盖前面的配置
* 命令参数
    - -f 强制日志轮转
    - -s 使用轮转状态文件，默认使用/var/lib/logrotate.status

# 默认轮转命令
/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf

# 配置选项
* rotate count：日志轮转数量【保留的旧日志数量】
* include：包含其他配置选项
    - 包含在全局配置中，但不能包含在log配置中
* compress：旧日志压缩，默认使用gzip
* delaycompress：延迟压缩功能
* nocompress：不对旧日志进行压缩
* copy：对当前日志文件创建一个副本，但不改变原来的日志文件【create选项会覆盖此功能】
* nocopy：不复制日志
* copytruncate：对当前日志建立一个副本后，清空原有日志文件【create选项会覆盖此功能】
    - 在实时日志较大时会出现小部分丢失清空
    - 适用于程序不能产生新日志文件的情况
* nocopytruncate：在建立新副本后不清空原有日志
* create mode owner group：设置新日志的属性
    - 需要搭配postrotate脚本来通知程序产生新的日志文件
    - 当任何的属性不设置时，都采用和原文件相同的属性
* nodelaycompress：不延迟压缩日志
* ifempty：日志为空也轮转
* notifempty：日志为空不轮转
* size：日志轮转大小，以字节为单位，可选单位【k、M、G】
* missingok：日志不存在不报错
* nomissingok：需要轮转的日志不存在时，报错
* daily：按天轮转日志
* weekly：按周轮转日志
* monthly：按月轮转日志
* yearly：按年轮转日志
* dateext：以日期设置旧日志文件后缀【布尔类型】
* nodateext：不用日期设置旧日志文件后缀
* dateformat：设置旧日志文件后缀格式【默认-%Y%m%d】
* olddir directory：旧日志被移动到一个单独的目录，可以是相对路径，也可以是绝对路径【但必须在同一个物理磁盘上】
* noolddir：轮转日志在同级目录
* postrotate/endscript：日志轮转之后执行的命令或脚本【/bin/sh】，这些指令只可能出现在日志文件定义中
    - 可以向进程发送特殊信号【signal】触发程序产生新的日志文件【相当于使用了create选项】
* prerotate/endscript：日志轮转之前执行的命令或脚本【/bin/sh】，这些指令只可能出现在日志文件定义中
* sharedscripts：对于同一个log定义下的所有待轮转的日志文件，他们所需执行的pre或post脚本仅被执行一次；
* nosharedscripts：默认情况下，同一个log定义下每个待轮转的文件都会触发pre或post脚本的执行

# 应用实践
```
# 全局设置
compress  

# 使用双引号以处理特殊情况，比如文件名中的空格等
# 可以在一行书写多个待轮转文件，空格分隔
"/var/log/httpd/access.log" /var/log/httpd/error.log { 
    xxxxxx
}

# 使用掩码进行文件名匹配
/var/log/news/* { 
    xxxxxx
}
```

[lograte]: https://linux.die.net/man/8/logrotate

