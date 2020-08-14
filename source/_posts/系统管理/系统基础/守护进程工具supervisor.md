---
title: 守护进程工具supervisor
tags:
  - supervisor
categories:
  - linux
date: 2018-05-30 17:10:25
---

# 介绍
supervisor用于管理非daemon进程

# 安装
* 可使用系统安装apt-get install supervisor
* 可使用python安装：pip install supervisor

# 配置文件
>更多配置说明请[参考][1]

```ini
[inet_http_server]  ;web管理界面设置
port=0.0.0.0:9001  
username=user      
password=123456    
[program:logstash] ;管理单个进程的配置
;设置需要守护的进程，此进程必须是前台执行
command=/home/zj-ops/logstash-5.1.1/bin/logstash -f /home/zj-ops/logstash-5.1.1/config/logstash-test.conf
;主要用于启动时间过长的进程
startsec=10
;是否随supervisor一起启动
priority=10
;不同服务的启动优先级，默认999
autostart=true
;异常退出后是否自动重启
autorestart=true
;运行进程的用户
user=zj-ops
;设置环境变量
environment=JAVA_HOME=/usr/local/jdk1.8.0_121
;日志配置，配合supervisor日志用于调试进程是否运行
redirect_stderr=True
stdout_logfile=/tmp/mgt_operation.log
stderr_logfile=/tmp/mgt_operation.log
```

# 配置范例
```
[program:mgt_user]
command=/home/{{ ansible_ssh_user }}/tomcat8/bin/catalina.sh run
environment=JAVA_HOME="/usr/local/jdk1.8.0_121"
startsec=100
directory=/home/{{ ansible_ssh_user }}
autostart=true
autorestart=true
user={{ ansible_ssh_user }}
```

# 命令行管理
>start、restart、stop都不会载入最新的配置文件。

* start xxx 启动某个进程  
* stop xxx 停止某个进程  
* status xxx 查看进程状态  
* restart xxx 重启某个进程  
* tail xxx 查看进程的日志  
* reload 载入最新的配置文件，停止原有进程并按新的配置启动、管理所有进程  
* update 根据最新的配置文件，启动新配置或有改动的进程，配置没有改动的进程不会受影响而重启。  

# web管理
```
port=0.0.0.0:9001  
username=user      
password=123456 
```

[1]:http://supervisord.org/configuration.html
