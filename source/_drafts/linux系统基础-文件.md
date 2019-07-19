---
title: linux系统基础-文件
tags:
categories:
---
# 文件类型
>以ls -l命令的输出的第一个符号为区别标志，其中字符设备、块设备、FIFO文件可使用mknod命令创建

* 普通文件：首字符显示为"-"，如：`-rw-rw-r-- 1 muker muker 135522 Jul 18 13:17 lsof.txt`【lsof.txt文件】
* 目录：首字符显示为“d”，如：`drwxrwxr-x  7 muker muker  4096 Jun 21 17:31 roles/`【roles目录】
* 符号链接：首字符显示为"l"，表示软连接，如：`lrwxrwxrwx. 1 root root 7 Oct 15  2017 /bin/python -> python2`
* 套接字文件：首字符显示为“s”，用于网络通信，如`srwx------ 1 root root 0 May 22 14:33 /var/run/supervisor/supervisor.sock`【supervisor程序的socket文件】
* 字符设备文件：首字符显示为”c“，表示串行端口设备，如：`crw--w---- 1 muker tty 136, 0 Jul 19 09:16 /dev/pts/0`【表示pts远程虚拟终端】
* 块设备文件：首字符显示为”b“，表示提供存储功能的设备，如：`brw-rw---- 1 root disk 253, 0 May 22 17:12 /dev/vda`【vda磁盘】
* FIFO文件(命名管道文件)：首字符显示为“p”，区别于“|”这种无名管道(管道运行完即销毁，同时也是单向通道)，FIFO这种方式为双向通道，任何程序任何时间都可以通过此管道进行双向通信

# 根目录
* sbin：放置设备管理命令，一般只能root账号运行
* bin：放置系统管理命令，一般用户可以使用
* dev：设备文件目录
* etc：配置文件目录、服务控制脚本
* boot：开机管理程序grub、内核kernel、initrd存放目录
* mnt/media：设备、镜像挂载目录
* lib/lib64：库文件目录
* home/root：用户家目录
* tmp：临时文件目录
* proc：系统运行产生的信息，不占用磁盘
* sys：与proc类似，主要记载与核心有关的信息，不占用磁盘
* opt/srv
* run
* usr：用户级配置信息
* var：存储内容会一直变化的内容，如log、pid等

# 目录
* 逻辑上，所有目录都挂载在根目录下
* 不同目录可以挂载在不同设备和分区

# 分区
由于开机过程只加载根分区，所有/bin:/sbin:/lib:/dev:/etc必须和根目录在同一个分区
