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
