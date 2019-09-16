---
title: 负载均衡器-lvs
tags:
  - lvs
categories:
  - linux
date: 2019-09-15 22:50:02
---

# 原理
[LVS][linuxvirtualserver]的全称是Linux virtual server，即Linux虚拟服务器。之所以是虚拟服务器，是因为LVS自身是个负载均衡器(director)，不直接处理请求，而是将请求转发至位于它后端真正的服务器realserver上。

## lvs技术实现
* 四层负载均衡：网络及传输层实现的IP虚拟服务器软件IPVS；ipvs是集成在内核中的框架，可以通过用户空间的程序ipvsadm工具来管理，该工具可以定义一些规则来管理内核中的ipvs，就像iptables和netfilter的关系一样。这也是本文介绍的功能。
* 七层负载均衡：基于内容请求分发的内核Layer-7交换机KTCPVS【功能尚不完善，使用的人不多】
* 集群管理软件：redhat公司从其6.1发行版起已包含LVS代码，他们开发了一个LVS集群管理工具叫Piranha，用于控制LVS集群，并提供了一个图形化的配置界面。【但此工具已被Redhat官方废弃，相应功能已使用[haproxy和keepalived][piranha]代替】

## ipvs技术术语
* LB：负载均衡器所在服务器
* RS：提供真实服务的服务器
* CIP（client IP）：客户端ip地址
* RIP（realserver IP）：真实提供服务的服务器ip地址
* DIP（director IP）：负载均衡器（director）上转发数据包到realserver的网卡地址
* VIP（virtual IP）：负载均衡器（director）上用于向客户端提供服务的ip地址

## ipvs的工作模式
当用户的请求到达负载调度器后，调度器如何将请求发送到提供服务的Real Server节点，而Real Server节点如何返回数据给用户，这正是IPVS实现的重点技术，IPVS实现负载均衡的机制有三种:

* VS/NAT（virtual server via network address translation）：通过网络地址转换将一组服务器构成一个高性能、高可用的虚拟服务器
* VS/TUN（vertual server via tunneling）：在分析VS/NAT的缺点和网络服务的非对称性的基础上提出的通过ip隧道实现虚拟服务器
* VS/DR（vertual server via director routing）：通过直接路由实现虚拟服务器
* FULLNAT：淘宝开源实现的模式，数据包进入LB时，同时对源地址和目标地址进行转换，从而实现RS可以跨VLAN通信，RS只需要连接内网，这样可以保证安全性。进站和出站的LB不一定是同一台机器。

### NAT模式
* 原理：通过网络地址转换，调度器LB重写请求报文的目标地址，根据预设的调度算法，将请求分派给后端的真实服务器，真实服务器的响应报文处理之后，返回时必须通过调度器（节点服务器的网关为负载均衡服务器），经过调度器时报文的源地址被重写，再返回给客户端，完成整个负载调度过程。
* 实现要点
    - 节点服务器需要将网关配置为LB的ip地址，这样才能保证报文经过LB，报文才能被改写
    - LB节点配置VIP和内核转发功能
    - nat模式支持对ip及端口的转换
* 优缺点：由于请求和响应的报文都经过LB【也就是像7层负载的处理方式一样，但却没有7层负载那么"多功能"】，而响应数据一般比请求数据大得多，LB的负载压力过大，调度器Director容易出现瓶颈

### TUN模式
* 原理：调度器把请求的报文通过ip隧道（将请求的报文封装在另一个ip报文中）转发至真实服务器，而真实服务器将处理后的响应报文直接返回给客户端。
* 实现要点：
    - LB和RS节点都需要配置ip隧道功能
    - LB和RS服务器lo网卡都要绑定VIP【防止真实网卡绑定vip造成ip冲突】，同时在RS上配置不对vip的arp广播请求响应，这样可以保证目标地址为vip的数据包只被director接受。
* 优缺点：
    - 调度器就只处理请求的入站报文。由于一般网络服务应答数据比请求报文大很多，采用此模式后，系统吞吐量可以提高10倍。
    - TUN模式的LAN环境转发不如DR模式效率高，而且还要考虑系统对ip隧道的支持问题
    - LAN环境一般多采用DR模式，WAN环境可以用TUN模式，但在当前WAN环境下，请求转发更多被haproxy、nginx、dns调度取代，因此应用不多。

### DR模式
* 原理：DR模式通过改写请求报文的目标MAC地址【请求地址仍为VIP】，将请求转发给真实的服务器，而真实服务器将响应后的处理结果直接返回给客户端。
* 实现要点
    - 要求调度器LB与真实服务器RS都有一块网卡连接在同一物理网段上。
    - 在LB和RS节点都需要配置VIP，并在RS配置抑制对vip的arp广播响应
    - 调度器不需要开启内核转发功能
    - RS与director的监听端口必须一致
* 优缺点：
    - 这种模式没有IP隧道的开销，对集群中的真实服务器也没有必须支持ip隧道协议的要求

### 模式对比
|      项目      |     vs/nat    |   vs/tun   |     vs/dr      |
|----------------|---------------|------------|----------------|
| server         | any           | Tnuneling  | Non-arp device |
| server network | private       | LAN/WAN    | LAN            |
| server number  | low           | High       | High           |
| server gateway | load balancer | own router | own router     |

* 在性能上，VS/DR和VS/TUN远高于VS/NAT，因为调度器只处于从客户到服务器的半连接中，按照半连接的TCP有限状态机进行状态迁移，极大程度上减轻了调度器的压力(真正建立TCP连接的是RS和Client)。VS/DR性能又稍高于VS/TUN，因为少了隧道的开销
* VS/DR和VS/TUN的主要区别是VS/TUN可以跨网络实现后端服务器负载均衡(也可以局域网内)，而VS/DR只能和director在局域网内进行负载均衡。

## ipvs调度算法
IPVS在内核中的负载均衡调度是以连接为粒度的。在HTTP协议（非持久）中，每个对象从WEB服务器上获取都需要建立一个TCP连接，同一用户 的不同请求会被调度到不同的服务器上，所以这种细粒度的调度在一定程度上可以避免单个用户访问的突发性引起服务器间的负载不平衡。在内核中的连接调度算法上，IPVS已实现了八种调度算法，其中，又根据运行方式不同，分为两大类：静态调度和动态反馈调度
### 静态调度
>不管RS的繁忙程度，根据调度算法计算后轮到谁就调度谁。

* 轮叫调度（Round-Robin Scheduling，rr）
* 加权轮叫调度（Weighted Round-Robin Scheduling，wrr），按照权重比例作为轮询标准
* 目标地址散列调度（Destination Hashing Scheduling，dh），目标地址哈希，对于同一目标IP的请求总是发往同一服务器；
* 源地址散列调度（Source Hashing Scheduling，sh），源地址哈希，在一定时间内，只要是来自同一个客户端的请求，就发送至同一个realserver。源地址散列调度和目标地址散列调度可以结合使用在防火墙集群中，它们可以保证整个系统的唯一出入口。

### 动态反馈调度
>根据RS的繁忙程度反馈，计算出下一个连接应该调度谁(动态反馈负载均衡算法考虑服务器的实时负载和响应情况，不断调整服务器间处理请求的比例，来避免有些服务器超载时依然收到大量请求，从而提高整个系统的吞吐率)。

* 最小连接调度（Least-Connection Scheduling，lc），调度器需要记录各个服务器已建立连接的数目，当一个请求被调度到某服务器，其连接数加1；当连接中止或超时，其连接数减1。当各个服务器的处理能力不同时，该算法不理想。
* 加权最小连接调度（Weighted Least-Connection Scheduling，wlc）：ipvs默认调度算法
* 基于本地的最少连接（Locality-Based Least Connections Scheduling，lblc）
* 带复制的基于局部性最少连接（Locality-Based Least Connections with Replication Scheduling，lblcr）：目前主要用于Cache集群系统；因为在Cache集群中客户请求报文的目标IP地址是变化的。

### 综合负载
计算算综合负载时，我们主要使用两大类负载信息：输入指标和服务器指标  
输入指标：主要是在单位时间内服务器收到新连接数与平均连接数的比例，它是在调度器上收集到的。  
服务器指标：  

*  主要记录服务器各种负载信息，如服务器当前CPU负载LOADi、服务器当前磁盘使用情况Di、当前内存利用情况Mi和当前进程数目 Pi。【有两种方法可以获得这些信息；一是在所有的服务器上运行着SNMP（Simple Network Management Protocol）服务进程，而在调度器上的Monitor Daemon通过SNMP向各个服务器查询获得这些信息；二是在服务器上实现和运行收集信息的Agent，由Agent定时地向Monitor Daemon报告负载信息。】
*  一个重要的服务器指标是服务器所提供服务的响应时间，它能比较好地反映服务器上请求等待队列的长度和请求的处理时间。调度器上的Monitor Daemon作为客户访问服务器所提供的服务，测得其响应时间。

### 权重计算
* 在实际使用中，若发现所有服务器的权值都小于他们的DEFAULT_WEIGHT，则说明整个服务器集群处于超载状态，这时需要加入新的服务器结点 到集群中来处理部分负载；
* 反之，若所有服务器的权值都接近于SCALE*DEFAULT_WEIGHT，则说明当前系统的负载都比较轻。

# 源码安装
>由于官方文档及网络均无ubuntu下源码安装参考，所以此处源码安装特指centos下的源码安装

* [lvs官方下载地址](http://www.linuxvirtualserver.org/software/ipvs.html)
* [最新版本ipvsadm](https://mirrors.edge.kernel.org/pub/linux/utils/kernel/ipvsadm/)

## 依赖安装
* 依赖安装：`yum install gcc* make kernel-devel libnl* popt* -y`
    - [编译安装参考](http://kb.linuxvirtualserver.org/wiki/Compiling_ipvsadm_on_different_Linux_distributions#Red_Hat_Enterprise_Linux_6)
* 设置内核软链接：ln -s /usr/src/kernels/$(uname -r) /usr/src/linux

## 编译安装
* tar xzf ipvsadm.tar.gz
* make 
* make install

## 安装结果测试
* 命令是否存在：ipvsadm
* 模块是否存在：lsmod |grep ip_vs

# 配置
准备1台LB和2台RS服务器；节点关闭防火墙和selinux
## LB节点设置
* LB主机添加vip地址：ifconfig eth0:0 192.168.100.10/24 up
* 设置tcp超时参数：ipvsadm --set 30 5 60
* LB添加vip及设置调度算法：ipvsadm -A -t 192.168.100.10:80 -s rr
* LB添加RS节点：
    - ipvsadm -a -t 192.168.100.10:80 -r 192.168.100.1:80 -g
    - ipvsadm -a -t 192.168.100.10:80 -r 192.168.100.2:80 -g

## RS节点设置
* lo网卡配置vip：ifconfig lo:1 192.168.100.10/32 up
* 添加路由【多网卡环境】：route add -host 192.168.100.10 eth0
* 设置节点抑制arp

```
echo 1 >/proc/sys/net/ipv4/conf/lo/arp_ignore 
echo 2 >/proc/sys/net/ipv4/conf/lo/arp_announce
echo 1 >/proc/sys/net/ipv4/conf/all/arp_ignore 
echo 2 >/proc/sys/net/ipv4/conf/all/arp_announce
```

## 状态查看
watch --interval=2 ipvsadm -Ln【watch持续监控LB状态】

## 其他操作命令
* ipvsadm -L                查看LB-director信息
* ipvsadm -C                  清空配置信息
* ipvsadm -d                   删除配置条目

## 控制脚本
* [DR模式-LB节点控制脚本](https://github.com/simple0426/sysadm/blob/master/shell/lvs/dr-lb.sh)
* [DR模式-RS节点控制脚本](https://github.com/simple0426/sysadm/blob/master/shell/lvs/dr-rs.sh)
* [TUN模式-RS节点控制脚本](https://github.com/simple0426/sysadm/blob/master/shell/lvs/tun-rs.sh)
* [主LB对RS节点执行健康检查](https://github.com/simple0426/sysadm/blob/master/shell/lvs/MasterLb-chk-rs.sh)
* [在备LB上实现对主LB的健康检查和接管](https://github.com/simple0426/sysadm/blob/master/shell/lvs/SlaveLb-chk-MasterLb.sh)【类似keepalived】

# 故障与排查
* 负载不均
    - 访问量较少，不均衡现象更加明显
    - 会话保持【lvs或后端服务】
    - 调度算法及用户请求的类型【时间长短、资源大小】
* 故障
    - 调度器上lvs调度规则及ip地址的正确性
    - 要在RS进行服务检查
    - RS节点绑定vip和抑制arp检查【对绑定的vip做实时监控并报警】
* 辅助排查工具tcpdump、ping

---

本文参考：https://www.cnblogs.com/wyzhou/p/9741790.html

[piranha]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/7.0_release_notes/sect-red_hat_enterprise_linux-7.0_release_notes-clustering-keepalived_and_haproxy_replace_piranha_as_load_balancer
[linuxvirtualserver]: http://www.linuxvirtualserver.org/zh/index.html