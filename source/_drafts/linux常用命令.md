---
title: linux常用命令
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

# 硬件信息
* 内存
    - free
* cpu
    - top
    - uptime
* 磁盘
    - iostat
    - df
* 网络
    - iftop

# 用户与权限
* chattr、chmod、chown
* useradd、userdel、usermod
* passwd
* setfacl、getfacl
* group、gpasswd、groups
* sudo、su

# 系统管理
* chkconfig
* crond
* date
* echo、head、tail、seq
* yum、apt、rpm、dpkg
* man
* xargs：把管道前面的处理结果按列表交给后面的命令处理
* reboot、init、shutdown、poweroff、halt
* history
* which、locate
* time：执行时间统计
* watch：周期性的执行程序，并同时全屏显示输出
* alias、unalias
* env

# 系统状态
* whoami：当前登录用户
* users：所有登录用户
* w、who：查看所有用户登录详情
* id
* last：查看系统最近登录情况
* lastlog：显示最近登录的用户名，登录端口及时间
* lastb：登录失败信息
* dmesg：系统启动过程
* uname
