---
title: docker网络
tags:
  - overlay
  - 网络命名空间
  - bridge
  - etcd
categories:
  - docker
date: 2019-12-02 23:51:37
---

# docker网络实现

* 网络命名空间(netns(net namespace))：linux在网络栈中引入网络名称空间，将独立的网络协议栈隔离到不同的名称空间中，彼此无法通信；docker利用linux的这一特性，实现了容器的网络隔离
* veth(virtual ethernet)：虚拟以太网卡，和物理网卡类似，用于不同命名空间的网络通信(例如，利用veth连接容器和宿主机)
* 网桥：网桥是一个二层网络设备，实现类似交换机那样的多对多通信（多个veth在同一个网桥中，实现多个容器间通信）

* iptables/netfilter：docker使用netfilter实现容器网络转发，即不同命名空间的流量转发

## 实现方式
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/docekr-bridge.jpg)

* 每个容器通过veth(虚拟以太网链接对)和宿主机连通
* 多个veth包含在一个bridge(桥接网络)中，因此多个容器可以互通
* 桥接网卡(一般为docker0)和宿主机出口(例如eth0)通过iptables的nat转换，从而可以让容器连接互联网

## docker网络类型

* bridge：实现网桥功能
* overlay：使用隧道方式连接不同宿主机上的容器
* none：容器绑定此网络后，除了回环接口外，容器没有其他网络地址
* host：容器绑定此网络后，容器直接使用宿主机的所有网络地址

## 跨主机容器互联

* 路由方式(underlay)：直接在宿主机配置到目标容器的路由

  >阿里云ecs需要在vpc上添加静态路由

  ```
  * 案例
    - 主机192.168.99.101  docker容器网络172.18.0.1/16
    - 主机192.168.99.102  docker容器网络172.17.0.1/16
  * 操作：
    - 101主机：route add -net 172.17.0.0/16 gw 192.168.99.102
    - 102主机：route add -net 172.18.0.0/16 gw 192.168.99.101
  ```

* 隧道方式(overlay)：使用隧道方式连接不同宿主机上的容器，docker也支持overlay网络

# linux网络命名空间

## 命名空间管理
* 查看命名空间：ip netns list
* 添加命名空间：ip netns add test
* 删除命名空间：ip netns delete test

## 网络接口配置
* 添加接口(一次添加网络互通的链接对(即两个接口))：ip link add veth-test1 type veth peer name veth-test2
* 将链接对的两端分配到命名空间中
    - ip link set veth-test1 netns test1
    - ip link set veth-test2 netns test2
* 添加ip地址：
    - ip netns exec test1 ip addr add 192.168.100.1/24 dev veth-test1
    - ip netns exec test2 ip addr add 192.168.100.2/24 dev veth-test2
* 启动接口
    - ip netns exec test1 ip link set dev veth-test1 up
    - ip netns exec test2 ip link set dev veth-test2 up

## 状态查询
* 查看端口状态：ip netns exec test2/test1 ip link 
* 查看ip地址：ip netns exec test2 ip addr
* 网络测试(test1中ping test2)：ip netns exec test1 ping 192.168.100.2

# docker网络管理-docker network

* 查看docker管理的网络列表(网卡)：docker network ls
* 查看网卡详情(容器、veth对等)：docker network inspect 4c44ab01e956

## bridge类型

* 创建桥接网卡：docker network create
  - 范例(指定网卡名称)：docker network create -d bridge my-bridge
  - 范例(指定网络地址)：docker network create --subnet 192.168.200.0/24 --gateway 192.168.200.1 my-bridge
* 创建容器并连接指定网络：docker run --network
  - 范例：docker run -idt --name test3 --network my-bridge busybox:latest /bin/sh -c "while true;do sleep 3600;done"
* 已存在的容器连接指定网络：docker network connect my-bridge test1

## overlay类型

### 简介

* 使用etcd分布式存储，docker创建的overlay网络可以实现跨主机容器互联
* 以下操作都是在双机(192.168.2.151,192.168.2.152)上运行，且需要先安装并配置etcd

### 安装并配置etcd

* 版本：注意etcd和docker engine版本兼容问题【如：docker engine 19.03.5和 flannel v0.11版本不支持etcd v3.4.3版本，但支持3.3.x系列】

* 下载并解压：https://github.com/etcd-io/etcd/releases

* 配置并启动：etcd --config-file etcd.yml

  ```
  # 节点名称
  name: "node1"
  # 存储位置
  data-dir: "/home/etcd"
  # 集群通信端口[本机]
  listen-peer-urls: "http://192.168.2.151:2380"
  # 集群服务端口[本机]
  listen-client-urls: "http://192.168.2.151:2379,http://127.0.0.1:2379"
  # 初始化时,告知其他节点本地集群通信端口[本机]
  initial-advertise-peer-urls: "http://192.168.2.151:2380"
  # 初始化时，发布的服务端口[本机]
  advertise-client-urls: "http://192.168.2.151:2379"
  # 新建集群
  initial-cluster-state: "new"
  # 集群标识
  initial-cluster-token: "etcd-cluster"
  # 集群节点
  initial-cluster: "node1=http://192.168.2.151:2380,node2=http://192.168.2.152:2380"
  ```

* 管理命令

  ```
  * 集群状态查看：ETCDCTL_API=3 etcdctl --endpoints=192.168.2.151:2379,192.168.2.152:2379 endpoint status/health
  ```

  

### docker-engine配置调整

```
{
    "registry-mirrors": ["https://2x97hcl1.mirror.aliyuncs.com"],
    "host": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"],
    "cluster-advertise": "192.168.2.151:2375",
    "cluster-store": "etcd://192.168.2.151:2379"
}
```
### 建立并使用overlay网络

* 创建网络：docker network create -d overlay demo
* 查看网络：docker network ls
* 在两台宿主机使用此网络启动：docker run -d --name test2 --network demo busybox sh -c "while true;do sleep 3600;done"
* 在两个容器内ping测试：docker exec -it test1 /bin/sh
* 在etcd中查看网络信息：etcdctl ls /docker

# 网桥管理工具-brctl

* 软件安装：bridge-utils【centos】
* 查看桥接网卡包含的容器链接对(veth)：brctl show 
* 添加桥接网卡
  + brctl addbr bridge0
  + ip addr add 172.19.0.1/24 dev bridge0
  + ip link set dev bridge0 up
