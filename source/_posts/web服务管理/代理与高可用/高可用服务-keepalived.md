---
title: 高可用服务-keepalived
date: 2019-09-16 11:32:43
tags:
    - keepalived
categories:
    - linux
---

# 简介
keepalived起初是专门为lvs设计的，专门用来监控lvs集群系统中各个服务节点的状态，后来又加入了VRRP（Virtual Router Redundancy Protocol）虚拟路由冗余协议。VRRP的出现是为解决静态路由出现的单点故障问题，他能保证网络的不间断、稳定的运行。所以keepalived有两个功能：

* 对lvs集群节点的健康检查【healthcheck】
* 对lvs集群LB节点的故障转移【failover】

## VRRP
>keepalived directors 之间故障转移时通过VRRP协议实现的。  

* 原理：在keepalived directors正常工作时，主director会不断的向备节点广播心跳消息，用以告诉备节点自己还活着；当主节点发生故障时，备节点就无法继续检测到主节点的心跳，进而调用自身的接管程序，接管主节点的ip资源和服务。而当主节点恢复故障时，备节点会释放自身接管的ip资源和服务，恢复到原来的备用角色
* vrrp协议要点
    - 通过竞选机制从backup中选出优先级最高的为master
    - 只有master有发送vrrp广播的权利
    - vrrp广播可以告诉节点更新arp表信息

# 安装
* [下载](https://www.keepalived.org/download.html)
* 依赖安装：yum install openssl openssl-devel -y
* 安装
    - tar xzvf keepalived-1.1.17.tar.gz
    - ./configure --sysconf=/etc
    - make && make install
* 规范化：ln -s /usr/local/sbin/keepalived /usr/sbin

# 配置
## 主配置文件
>即：keepalived.conf

```
global_defs {  # 全局定义部分
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id lvs_master #运行keepalived服务的服务器标识，主要用于发送邮件时显示
}

vrrp_instance VI_1 { # vrrp实例定义部分
    state MASTER # 定义keepalived的角色：MASTER表示主服务器，BACKUP表示备用服务器
    interface eth0 # 定义HA集群检测的服务接口
    lvs_sync_daemon_interface eth1 # 定义HA集群检测的心跳接口【默认使用服务接口】
    virtual_router_id 60 # 虚拟路由器标识，一个vrrp实例下的主备必须相同，不同vrrp实例的标识不同
    priority 100 # 一个实例中主服务器优先级高于备用服务器
    advert_int 1 # 心跳间隔
    authentication {
        auth_type PASS # 主备之间的认证类型
        auth_pass 1234 # 主备之间的认证密码
    }
    virtual_ipaddress {
        192.168.100.120 #vip地址
    }
}

virtual_server 192.168.100.120 80 { # 虚拟服务器部分，虚拟服务器的ip和端口
    delay_loop 6 # director的健康检查间隔
    lb_algo wrr # 负载调度算法
    lb_kind DR # 实现负载均衡的方式
    persistence_timeout 50 # 会话保持时间
    protocol TCP # director向RS转发的协议类型
    real_server 192.168.100.111 80 { # 真实服务器的ip和端口
    weight 1 # 服务器在被调度时的权重
    TCP_CHECK { # director对real-server的检测类型
         connect_timeout 3 # 连接超时时间
         nb_get_retry 3  # 重试次数
         delay_before_retry 3  # 重试间隔
    }
    }
    real_server 192.168.100.112 80 {
    weight 1
    TCP_CHECK { 
         connect_timeout 3
         nb_get_retry 3  
         delay_before_retry 3  
    }
    }
}
```
## 服务启动参数
>/etc/sysconfig/keepalived【centos】或/etc/default/keepalived【ubuntu】

```
# Options for keepalived. See `keepalived --help' output and keepalived(8) and
# keepalived.conf(5) man pages for a list of all options. Here are the most
# common ones :
#
# --vrrp               -P    Only run with VRRP subsystem.
# --check              -C    Only run with Health-checker subsystem.
# --dont-release-vrrp  -V    Dont remove VRRP VIPs & VROUTEs on daemon stop.
# --dont-release-ipvs  -I    Dont remove IPVS topology on daemon stop.
# --dump-conf          -d    Dump the configuration data.
# --log-detail         -D    Detailed log messages.
# --log-facility       -S    0-7 Set local syslog facility (default=LOG_DAEMON)

KEEPALIVED_OPTIONS="-D -d -S 0"
```
## 日志设置
>设置系统syslog

* 配置设置：echo "local0.*   /var/log/keepalived.log" >> /etc/rsyslog.conf
* 重启syslog服务：/etc/init.d/rsyslog restart

# 服务控制
* 启动服务：/etc/init.d/keepalived start
* 开机自启：chkconfig keepalived on
