---
title: docker网络
tags:
categories:
---
# linux网络命令空间
## 命名空间管理
>netns(net name space)：网络命名空间

* 查看命名空间：ip netns list
* 添加命名空间：ip netns add test
* 删除命名空间：ip netns delete test

## 端口和地址操作
>veth(virtual ethernet)：虚拟以太网卡，和物理网卡类似，用于网络连通

* 添加端口(一次添加网络互通的链接对(即两个端口))：ip link add veth-test1 type veth peer name veth-test2
* 将链接对的两端分配到命名空间中
    - ip link set veth-test1 netns test1
    - ip link set veth-test2 netns test2
* 添加ip地址：
    - ip netns exec test1 ip addr add 192.168.100.1/24 dev veth-test1
    - ip netns exec test2 ip addr add 192.168.100.2/24 dev veth-test2
* 启动端口
    - ip netns exec test1 ip link set dev veth-test1 up
    - ip netns exec test2 ip link set dev veth-test2 up

## 状态查询
* 查看端口状态：ip netns exec test2/test1 ip link 
* 查看ip地址：ip netns exec test2 ip addr
* 网络测试(test1中ping test2)：ip netns exec test1 ping 192.168.100.2

# 桥接网络-bridge
## 互通原理
* 每个容器通过veth(虚拟以太网链接对)和宿主机连通
* 多个veth包含在一个bridge(桥接网络)中，因此多个容器可以互通
* 桥接网卡(一般为docker)和宿主机出口(例如eth0)通过iptables的nat转换，从而可以让容器连接互联网

## 操作
### docker-network命令
* 查看桥接网卡列表：docker network ls
* 查看桥接网卡详情(容器、veth对等)：docker network inspect 4c44ab01e956
* 创建桥接网卡：docker network create
    - 范例(指定网卡名称)：docker network create -d bridge my-bridge
    - 范例(指定网络地址)：docker network create --subnet 192.168.200.0/24 --gateway 192.168.200.1 my-bridge
* 创建容器并连接指定网络：docker run --network
    - 范例：docker run -idt --name test3 --network my-bridge busybox:latest /bin/sh -c "while true;do sleep 3600;done"
* 已存在的容器连接指定网络：docker network connect my-bridge test1

### brctl命令
* 查看桥接网卡包含的容器链接对(veth)：brctl show 
* 添加桥接网卡
    + brctl addbr bridge0
    + ip addr add 172.19.0.1/24 dev bridge0
    + ip link set dev bridge0 up

## 容器端口映射
docker run -p host_port:container_port

## 特殊网络
>docker network ls看到的网络

* none：容器绑定此网络后，除了回环接口外，容器没有其他网络地址
* host：容器绑定此网络后，容器直接使用宿主机的所有网络地址
