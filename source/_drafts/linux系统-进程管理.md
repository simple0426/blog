---
title: linux系统-进程管理
tags:
categories:
---
# 进程
* kill、pkill、killall
* ps
* fuser
* nohup
* jobs
* fg
* [lsof](#lsof)

# lsof
## 简介
list open file  列出打开的文件，文件可以是普通文件、目录、链接、字符设备、块设备、FIFO设备、socket
## 选项
>没有任何选项时，lsof默认显示所有活跃进程打开的文件

* `-a`：指明所有选项之间逻辑“与”关系【默认“或”关系】
* -c `^` c：进程执行的命令(不)是以c开头的，c可以是正则表达式【以`//`进行包围】
* -r t：周期性的显示输出信息【t定义时间间隔，默认15s】
* -u `^` uid：(不)属于某个登录名或id的用户，多个用户之间逗号分隔
* -p `^` pid：(不)属于某个进程的
* -i 地址：选择与internet地址匹配的文件列表：
    - 地址：`[46][protocol][@hostname|hostaddr][:service|port]`
    - protocol：传输层协议tcp/udp
    - hostname、hostaddr：主机名、ip地址
    - service：应用层协议(如smtp)、端口号
    - 范例：lsof -i tcp@ops:ssh
* -t：只输出符合要求的进程pid
* -X：不关联TCP、UDP的文件
    - 范例：lsof -p 22929 -X
* -s `^` `[p:s]`：(不)属于某个tcp、udp协议状态(如established)的
    - 范例：lsof -a -i tcp -s tcp:established -c nginx【查看nginx并发连接】
* --：表示选项的的结束，在文件名中包含“-”时有用
* file-names：列出打开文件的进程
