---
title: linux命令-文件管理
tags:
categories:
---
# 文件管理
* cat
* ls、cd、pwd
* du
* mount、parted
* cp、mv、rm
* touch、mkdir、rename
* rename
* tar、gzip
* tree
* file、stat
* dos2unix/unix2dos
* fsck、e2fsck
* dd
* diff、vimdiff

# lsof
* lsof：list open file  列出打开的文件
* 文件可以是普通文件、目录、链接、字符设备、块设备、FIFO设备、socket
* 没有任何选项时，lsof默认显示所有活跃进程打开的文件
* 多个选项直接为逻辑“或”关系

## 选项
* `-a`选项指明所有选项之间逻辑“与”关系
* -c `^` c：选择进程执行的命令(不)是以c开头的，c可以是正则表达式
* -F：指定输出格式【以便其他程序可以处理】
* +|-r  `[t[m<fmt>]]`：周期性的显示输出信息
    - fmt：strftime格式时间【未定义时，使用==分隔周期输出】
    - t定义时间间隔，默认15s
* -u `^` uid：(不)属于某个登录名或id的用户
* -p `^` pid：(不)属于某个进程的
* -s `^` `[p:s]`：(不)属于某个tcp、udp协议状态(如established)的
    - 范例：lsof -a -i tcp -s tcp:established -c nginx【查看nginx并发连接】
* -i 地址：选择与internet地址匹配的文件列表：
    - 地址：`[46][protocol][@hostname|hostaddr][:service|port]`
    - protocol：传输层协议tcp/udp
    - hostname：主机名
    - hostaddr：IP地址
    - service：应用层协议，如smtp
    - port：端口号
    - 范例：lsof -i tcp@ops:ssh
* -t：只输出符合要求的进程pid
