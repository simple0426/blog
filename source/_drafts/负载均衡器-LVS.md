---
title: 负载均衡器-LVS
tags:
categories:
---
# 原理
* LVS的全称是Linux virtual server，即Linux虚拟服务器。之所以是虚拟服务器，是因为LVS自身是个负载均衡器(director)，不直接处理请求，而是将请求转发至位于它后端真正的服务器realserver上。
* ipvs是集成在内核中的框架，可以通过用户空间的程序ipvsadm工具来管理，该工具可以定义一些规则来管理内核中的ipvs。就像iptables和netfilter的关系一样。

## 负载均衡技术
* 基于DNS的A记录负载均衡
* 基于客户端的负载均衡实现：客户端存储服务器集群配置信息，当有请求时，通过算法向不同服务器发送请求【redis的集群即使用此方法，redis的客户端需要存储redis集群的ip及端口列表】
* 基于应用层（七层）负载均衡实现：当用户访问请求到达调度器时，请求会提交给负载均衡调度的应用程 序分析请求，根据各个服务器的负载情况，选出一台服务器，重写请求并向选出的服务器访问，取得结果后，再返回给用户。
* 基于传输层（四层）负载均衡实现：在tcp/ip级别上实现负载均衡

## 技术实现
* 四层负载均衡【ipvs】：传输层实现的IP虚拟服务器软件IPVS，在IP负载均衡技术中，需要服务器池拥有相同的内容提供相同的服务。
* 七层负载均衡【ktcpvs】：基于内容请求分发的内核Layer-7交 换机KTCPVS(功能尚不完善，使用的人不多)【在基于内容请求分发技术中，服务器可以提供不同的服务，当客户请求到达时，调度 器可根据请求的内容选择服务器 执行请求】
* 集群管理软件：redhat公司在从其6.1发行版起已包含LVS代码，他们开发了一个LVS集 群管理工具叫Piranha，用于控制LVS集群，并提供了一个图形化的配置界面。【但此工具已被Redhat官方废弃，相应功能已使用[haproxy和keepalived][piranha]代替】

# 源码安装
由于官方文档及网络均无ubuntu下源码安装参考，所以此处源码安装特指centos下的源码安装
## 下载地址
* lvs官方下载地址：http://www.linuxvirtualserver.org/software/ipvs.html
* 最新版本ipvsadm：https://mirrors.edge.kernel.org/pub/linux/utils/kernel/ipvsadm/
* centos系列编译安装参考：http://kb.linuxvirtualserver.org/wiki/Compiling_ipvsadm_on_different_Linux_distributions#Red_Hat_Enterprise_Linux_6

## 依赖安装
* 安装内核源码【lvs使用内核源码里的头文件,即nclude目录】
    - 安装：yum install kernel-devel -y
    - 设置软链接：ln -s /usr/src/kernels/2.6.32-358.el6.x86_64 /usr/src/linux
* 安装依赖：yum install libnl* popt* -y

## 编译安装
* tar xzf 
* make 
* make install

## 测试
* 命令是否存在：ipvsadm
* 模块是否存在：lsmod |grep ip_vs

[piranha]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/7.0_release_notes/sect-red_hat_enterprise_linux-7.0_release_notes-clustering-keepalived_and_haproxy_replace_piranha_as_load_balancer