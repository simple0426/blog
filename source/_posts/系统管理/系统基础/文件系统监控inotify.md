---
title: 文件系统监控inotify
tags:
  - inotify
categories:
  - linux
date: 2019-05-05 22:21:40
---

# 简介
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
统计文件和目录发生的变化
## 参数
* -z --zero 输出表格的行和列，即使元素为空
* --exclude 正则匹配需要排除的文件，大小写敏感
* --excludei 正则匹配需要排除的文件，大小写忽略
* -r --recursie 监控一个目录下的所有子目录
* -t --timeout 设置超时时间
* -e --event `<event1>`监控指定的事件，多个事件之间逗号分隔
*  -a|--ascending `<event>` 升序统计指定事件
*  -d|--descending `<event>` 降序统计指定事件

## 范例
inotifywatch -v -e access -e modify -t 60 -r /home
# inotifywait
等待文件或目录发生的变化
## 参数
* -m|--monitor 接收一个事件而不退出，无限执行下去。默认接受一个事件后退出
* -d|--daemon 和-m一样，但是在后台执行，同时必须指定--outfile选项，包含--syslog
* -o|--outfile `<file>` 输出事件到文件
* -s|--syslog 输出错误到系统日志
* -r --recursie 监控一个目录下的所有子目录
* -q|--quiet 不输出详细信息（指定2次时，除了致命错误，不会输出任何信息)
* --exclude 正则匹配需要排除的文件，大小写敏感
* --excludei 正则匹配需要排除的文件，大小写忽略
* -t --timeout 接收一个事件前的超时退出时间，0为永不超时；但是接收一个事件后会退出
* -e --event `<event1>`监控指定的事件，多个事件之间逗号分隔
* -c|--csv 输出csv格式
* --timefmt `<fmt>` 指定时间格式
    - --timefmt '%y-%m-%d %H:%M'
* --format `<fmt>` 指定输出信息格式
    - --format '%T %f %e'
    - %w 发生事件的目录
    - %f 发生事件的文件
    - %e 发生的事件
    - %T timefmt定义的时间格式

## 范例
* 监控事件：inotifywait -mrq -e 'create,delete,close_write,attrib,moved_to' --timefmt '%Y-%m-%d %H:%M' --format '%T %f %e' /tmp/
* 排除特定的目录和文件不予监控：`inotifywait -mrq --exclude ".*\.py|test.*/\..*" -e "open,access,modify" python_scripts/`【此例为排除python_scripts下的py文件和test**目录下的隐藏文件】

```
2018-05-21 19:53 xiaoke.txt CREATE  
2018-05-21 19:53 xiaoke.txt ATTRIB  
2018-05-21 19:53 xiaoke.txt CLOSE_WRITE,CLOSE  
2018-05-21 19:54 xiaoke.txt DELETE  
```

# rsync+inotify
可实现单向的目录或文件的实时同步
## 使用步骤
* 安装并配置rsync【daemon模式服务端与客户端设置】
* 安装inotify
* 编写实时同步脚本

```
#!/bin/bash
src=/oldboy
dst=oldboy
user=backup_user
host=192.168.1.1
inotifywait -mrq --timefmt '%d/%m/%y %H:%M' --format '%T %w%f%e' -e modify,delete,create,attrib $src \
| while read files
        do
        cd $src && rsync -arzu -R --delete --timeout=100 --password-file=/etc/rsyncd.pass ./ $user@$host::$dst  
done
```

* 将同步脚本放入后台执行【nohup cmd &或supervisor方式】

## 使用注意
* rsync的客户端和服务端的密码文件内容不同
    - 服务端含用户名和密码
    - 客户端只有密码
* 服务端和客户端的密码文件权限相似
    - 属主只能是运行用户
    - 权限只能是0600
* rsync-daemon端要关闭防火墙和selinux
* 若要同步目录的内容一定要进入目录后再进行同步，否则会将客户端的目录作为服务端模块下的子目录进行同步
