---
title: linux实用命令
date: 2018-04-16 10:15:24
tags: ['技巧']
categories: ['linux']
---

# nc
>netcat是一个用于TCP/UDP连接和监听的linux工具,nc是开源版本的netcat工具
## 服务端
监听端口：nc -l 8888
## 客户端
向服务端发送数据：nc 127.0.0.1 8888 < file.txt

# socat
>类似于netcat的工具，名字来由是” Socket CAT”，可以看作是netcat的N倍加强版  
>主要特点就是在两个数据流之间建立通道
## haproxy应用
* help信息：echo ""|sudo socat stdio /var/run/haproxy.sock
* 状态查看：echo "show stat"|sudo socat stdio /var/run/haproxy.sock
* 启停server：echo "enable server service_marketinfo/172.17.134.60"|sudo socat stdio /var/run/haproxy.sock

# logger
>在shell命令行向syslog打入指定的信息

## 范例
echo "this is message"|logger -it logger_test -p local3.notice

* -i 在每行都记录进程ID
* -t logger_test 每行记录都加上“logger_test”这个标签
* -p local3.notice 设置记录的设备和级别

# wget
>下载工具

## 可用参数
* -c 支持断点续传
* --limit-rate 限速【1M=限速1MB/s】

## 范例
wget -c --limit-rate=1M https://mirrors.aliyun.com/deepin-cd/15.3/deepin-15.3-amd64.iso
