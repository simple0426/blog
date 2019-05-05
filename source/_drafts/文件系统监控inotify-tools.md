---
title: 文件系统监控inotify-tools
date: 2019-05-05 22:21:40
tags:
    - inotify
categories:
    - linux
---
# 设计目的
inotify是一个细粒度的异步文件监控系统，可以监控文件发生的一切变化，如：

* 访问属性
* 读写属性
* 权限属性
* 删除
* 移动

inotify-tools是linux下使用inotify接口的简单实现
# 设计原理
# 安装步骤
## 检查内核是否支持inotify
`ls -l /proc/sys/fs/inotify/`
## 安装
* 【centos】：yum install inotify-tools -y
* 【ubuntu】：apt install inotify-tools -y

## 检查
安装完成后会得到两个命令:inotifywait和inotifywatch

# 命令详解

# 常用范例
# 常见问题
