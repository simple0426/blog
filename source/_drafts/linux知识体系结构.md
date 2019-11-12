---
title: linux知识体系结构
tags:
categories:
---
# 文件管理
* 磁盘
    - raid
* 分区
    - 磁盘配额【废弃】
    - LVM
* 格式化
* 文件系统
    - drbd【distribute replication block device】：分布式块设备同步
    - nfs【废弃】
    - autofs【废弃】
    - 分布式文件系统
        + hdfs
        + ceph
* 系统启动

# 权限管理
* 用户
* 权限
    - ldap【待定】

# 服务管理【运行状态】
* 软件安装与服务启停控制
* 进程管理
* cpu、磁盘、内存、网络状态
* 登录状态
* 故障诊断
* free：显示内存【内存状态】
    - -m：以M为单位
    - -h：以友好的单位
* uptime：显示系统负载
* iostat【磁盘状态】
    - tps：每秒钟收到的io请求数
    - kB_read/s：每秒钟读的字节数
    - kB_wrtn/s：每秒钟写的字节数
    - kB_read：读取字节总数
    - kB_wrtn：写字节总数
* iftop

# shell编程和文本处理

# 监控与报警
* nagios+cacti【废弃】
* zabbix
* open-falcon【重点】

# 辅助开发及部署
* 跨互联网站点间vpn
* ftp
* samba
* dns【bind9】
* git
* vpn

# 缓存辨析
* xcache(php)：Xcache 是 php 底层的缓存，它将PHP程式编译成字节码（byte code），再透过服务器上安装对应的程式来执行PHP脚本；xcache 是不需要修改PHP程序的，只要安装了就可以自动为你的程序加速
* memcached：memcached 是应用层缓存，它通过在内存中缓存数据和对象来减少读取数据库的次数，从而提高动态、数据库驱动网站的速度；memcached 则需要你修改程序的，需要你在操作数据库之前先询问下 memcached 有没有缓存数据，如果有且没有过期则不再访问数据库，以达到减少数据库查询的目的。
