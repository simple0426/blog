---
title: 负载均衡器-haproxy
tags:
  - haproxy
categories:
  - linux
date: 2019-09-10 10:56:44
---

# 软件安装
* yum或apt方式安装
* 源码安装

```
make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 
make install
```

# 配置
## 主配置
haproxy.cfg详细配置，请[查看][haproxy.cfg] ！！！

* global：全局设置，通常和操作系统相关，如进程
* defaults：默认配置，可以被frontend、backend、listen部分继承
* fontend：用来接收客户端请求，根据域名、uri等定义不同的匹配规则，并将请求转发给不同的backend
* backend：定义后端服务器集群，并设置后端服务器的权重、连接超时、调度、健康检查等
* listen：可以理解为backend和fontend的组合体

## 日志设置
>haproxy使用系统syslog的方式写日志【udp协议，514端口】

```
$ModLoad imudp
$UDPServerRun 514
local0.*                                           /usr/local/haproxy/logs/haproxy.log
```

# 服务控制
## 服务控制
* 重载配置：service haproxy reload
* 启动、停止、重启：start/stop/restart

## haproxy命令选项
* -f：指定配置文件
* -c：检测配置文件语法
* -D：守护进程模式
* -p pid：指定启动后的pid文件
* `-sf/-st [pid ]*`：终止旧的pid【hard、soft】

## socket命令
* 前置要求：在配置文件haproxy.cfg的global区段配置【stats socket /var/run/haproxy.sock mode 600 level admin】
* 作用：连接socket后，管理负载均衡器代理的前后端服务器【fontend、backend】
* 工具：socat是类似于netcat的工具，名字来由是Socket CAT，可以看作是netcat的N倍加强版，主要特点就是在两个数据流之间建立通道
* socat用法：
    - help信息：echo ""|sudo socat stdio /var/run/haproxy.sock
    - 状态查看：echo "show stat"|sudo socat stdio /var/run/haproxy.sock
    - 启停server：echo "enable server service_marketinfo/172.17.134.60"|sudo socat stdio /var/run/haproxy.sock

[haproxy.cfg]: https://github.com/simple0426/sysadm/blob/master/config/web/haproxy.cfg