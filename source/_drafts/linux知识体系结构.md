---
title: linux知识体系结构
tags:
categories:
---
# 文件管理
* 磁盘
* 分区
* 格式化
* 文件系统
    - nfs【废弃】
    - autofs【废弃】
    - 分布式文件系统
        + hdfs
        + ceph
* 系统启动

# 权限管理
* 用户
* 权限

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

# vpn分类
* 远程拨入办公区
* 站点之间网络互连
* 翻墙代理类

# 辅助开发服务
* ftp
* samba
* bind
* git
* vpn
