---
title: linux日志切割logrotate
tags: logrotate
categories: linux
---
# 介绍
* 功能
    - 日志轮转
    - 压缩
    - 删除
* 默认在每天的定时任务【cron.daily】下执行，因此每天只会执行一次
* 只有当日志轮转参数基于日志大小并且多次运行logrotate，或者使用-f参数【logrotate -f】才会形成对日志的修改
* 可以在命令行使用任意多的配置文件或定义配置文件目录；具有相同功能的配置选项，后来的配置会覆盖前面的配置
* 命令参数
    - -f 强制日志轮转
    - -s 使用轮转状态文件，默认/var/lib/logrotate.status

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
* nodelaycompress：不延迟压缩日志
* ifempty：日志为空也轮转
* notifempty：日志为空不轮转
* minsize：在daily, weekly, monthly, or yearly轮转周期下达到此值也进行轮转
* size：日志轮转大小，以字节为单位，可选单位【k、M、G】
    - 此选项与daily, weekly, monthly, or yearly轮转周期互不影响独立运转
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
* postrotate/endscript：日志轮转之后执行的命令或脚本【/bin/sh】，这些指令可能只出现在日志文件定义中
* prerotate/endscript：日志轮转之前执行的命令或脚本【/bin/sh】，这些指令可能只出现在日志文件定义中

# 实践
```
compress  #全局设置

# 使用双引号以处理特殊情况，比如文件名中的空格等
# 可以在一行书写多个轮转选项，以空格分隔
"/var/log/httpd/access.log" /var/log/httpd/error.log { 
    xxxxxx
}

#使用掩码进行文件名匹配
/var/log/news/* { 
    xxxxxx
}
```

# 参考
* 官方参考：https://linux.die.net/man/8/logrotate
* 参考2：https://blog.51cto.com/istyle/1785003
