---
title: 文件系统监控inotify-tools
tags:
  - inotify
categories:
  - linux
date: 2019-05-05 22:21:40
---

# inotify
inotify是一个细粒度的异步文件监控系统，可以监控文件发生的一切变化，如：

* 访问属性
* 读写属性
* 权限属性
* 删除
* 移动

inotify-tools是linux下使用inotify接口的简单实现
# 安装
## 内核检查
`ls -l /proc/sys/fs/inotify/`
## 安装
* 【centos】：yum install inotify-tools -y
* 【ubuntu】：apt install inotify-tools -y

## 安装检查
安装完成后会得到两个命令:inotifywait和inotifywatch

# inotify事件
|  事件  |      详解      |
|--------|----------------|
| access | 访问或读取     |
| modify | 内容变更       |
| attrib | 属性变更       |
| close  | 文件或目录关闭 |
| open   | 文件或目录打开 |
| move   | 文件或目录移动 |
| create | 新建文件或目录 |
| delete | 文件或目录删除 |

# inotifywatch
>统计文件和目录发生的变化
## 参数
* @file 排除指定的文件不监控，可以绝对路径，也可以是相对路径
* --fromfile 从文件中读取要监控的文件，一个文件一行，排除的文件已@开头
* -z --zero 输出表格的行和列，即使元素为空
* --exclude 正则匹配需要排除的文件，大小写敏感
* --excludei 正则匹配需要排除的文件，大小写忽略
* -r --recursie 监控一个目录下的所有子目录
* -t --timeout 设置超时时间
* -e --event <event1>监控指定的事件，多个事件之间逗号分隔
*  -a|--ascending <event> 升序统计指定事件
*  -d|--descending <event> 降序统计指定事件

## 范例
inotifywatch -v -e access -e modify -t 60 -r /home
# inotifywait
>等待文件或目录发生的变化
## 参数
* @file 排除指定的文件不监控，可以绝对路径，也可以是相对路径
* --fromfile 从文件中读取要监控的文件，一个文件一行，排除的文件已@开头
* -m|--monitor 接收一个事件而不退出，无限执行下去。默认接受一个事件后退出
* -d|--daemon 和-m一样，但是在后台执行，同时必须指定--outfile选项，包含--syslog
* -o|--outfile <file> 输出事件到文件
* -s|--syslog 输出错误到系统日志
* -r --recursie 监控一个目录下的所有子目录
* -q|--quiet 指定1次，不会输出详细信息；指定2次，除了致命错误，不会输出任何信息
* --exclude 正则匹配需要排除的文件，大小写敏感
* --excludei 正则匹配需要排除的文件，大小写忽略
* -t --timeout 接收一个事件前的超时退出时间，0为永不超时；但是接收一个事件后会退出
* -e --event <event1>监控指定的事件，多个事件之间逗号分隔
* -c|--csv 输出csv格式
* --timefmt <fmt> 指定时间格式
    - --timefmt '%y-%m-%d %H:%M'
* --format <fmt> 指定输出信息格式
    - --format '%T %f %e'
    - %w 发生事件的目录
    - %f 发生事件的文件
    - %e 发生的事件
    - %T timefmt定义的时间格式

## 范例
inotifywait -mrq -e 'create,delete,close_write,attrib,moved_to' --timefmt '%Y-%m-%d %H:%M' --format '%T %f %e' /tmp/

>2018-05-21 19:53 xiaoke.txt CREATE  
>2018-05-21 19:53 xiaoke.txt ATTRIB  
>2018-05-21 19:53 xiaoke.txt CLOSE_WRITE,CLOSE  
>2018-05-21 19:54 xiaoke.txt DELETE  

