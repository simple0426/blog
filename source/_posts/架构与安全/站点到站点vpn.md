---
title: 站点到站点vpn
date: 2019-09-17 15:05:35
tags: ['vpn']
categories: ['linux']
---
# 实现目标
* IDC服务器和AWS云服务器在都只有一个公网出口的情况下【即：服务器只有内网ip，通过nat转换访问公网】实现内网ip互通【即：IDC服务器和AWS服务器可以ping通对方的内网ip地址】
* 使用openvpn软件架设穿透公网的vpn隧道

# 网络规划
## [实现要点](https://boke.wsfnk.com/archives/699.html)
- 在网关设备上采用SNAT，使内部网络可上互联网（跟普通家用wfi一个原理）
- 在网关设备上采用DNAT，将其wan address 的指定端口映射到内部openvpn-server的指定端口，(默认udp1194)
- 引流，在网关设备上将去往对端内网的流量，重定向到openvpn-server

## IDC机房
* 使用华为AR2240作为出口网关
    - 内网1:10.86.0.0/24【作为主要内网，通过地址映射或地址转换实现内部服务器的联网】
    - 内网2: 172.16.10.0/24【辅助内网，增加网络冗余】
* 选择一台内网服务器作为openvpn服务的宿主机【为了保证两端vpn服务可以建立连接，openvpn的服务端口应该被发布到公网】

## AWS云
* 使用10.0.0.0/24作为内网
* 任意选择一台EC2作为代理服务器【此代理服务器应当具有外网ip】，代理内网服务器的Internet联网
* 另选一台EC2作为openvpn服务宿主机【为了保证两端vpn服务可以建立连接，openvpn的服务端口应该被发布到公网】

# 内网服务器访问公网
>在只保留一个公网出口的情况下，内网服务器可以访问互联网

## IDC机房
* 需要有公网ip的服务器通过地址映射将公网ip留在路由器上
* 不需要公网ip的服务器，只配置默认网关即可

## AWS云
* 在aws上建立vpc，内网规划为10.0.0.0/24
* 在aws新建2个ec2，地址分别为10.0.0.60、10.0.0.61，申请弹性ip并绑定至61
* 在61上开启代理【作为代理服务器】
    - sysctl -w net.ipv4.ip_forward=1
    - iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
* 将其他ec2的默认网关更改为10.0.0.61
    - route del default gw 10.0.0.1
    - route add default gw 10.0.0.61

# 架设vpn隧道
## AWS云
* 将vpn服务端口发布到公网：iptables -t nat -A PREROUTING -i eth0 -d 1.1.1.1/32 -p tcp --dport port 8000 -j DNAT --to-destination 10.0.0.60:8000
* 生成用于安全认证的静态key文件：openvpn --genkey --secret /etc/openvpn/vpn.key
* 开启路由转发：sysctl -w net.ipv4.ip_forward=1
* 在60上开启vpn服务，配置如下：

```
remote 42.62.65.23        #对端公网ip
float
port 8000
dev tun
ifconfig 10.10.0.1 10.10.0.2  #设置本端和对端的隧道地址
persist-tun
persist-key
persist-local-ip
persist-remote-ip
comp-lzo                 #开启压缩
secret /etc/openvpn/vpn.key
route 10.86.0.0 255.255.255.0  #设置到对端私网的路由
route 172.16.10.0 255.255.255.0
route 192.168.10.0 255.255.255.0
route 10.150.10.0 255.255.255.0
chroot /var/empty
user vpn
group vpn
log /var/log/vpn.log         #配置日志
verb 3
```

* 为了保证网络的可用性，使用supervisor守护openvpn服务

## IDC机房
* 将vpn服务端口发布到公网
* 拷贝在aws生成的key文件到本地
* 余下操作参照aws上的操作即可

# 路由设置
在网关设备上设置路由，将到对方内网的流量转发到vpn隧道所在服务器上
## AWS云
>在代理服务器61上设置

```
route add -net 10.86.0.0 netmask 255.255.255.0 gw 10.0.0.60
route add -net 172.16.10.0 netmask 255.255.255.0 gw 10.0.0.60
route add -net 10.10.0.0 netmask 255.255.255.0 gw 10.0.0.60
```
## IDC机房
>在路由器AR2240设置静态路由【类命令如下】

```
route add -net 10.0.0.0 netmask 255.255.255.0 gw 10.86.0.103
route add -net 10.10.0.0 netmask 255.255.255.0 gw 10.86.0.103
```

