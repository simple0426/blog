---
title: docker网络
tags:
  - overlay
  - 网络命名空间
  - bridge
  - etcd
  - ovs
categories:
  - 虚拟化
date: 2019-12-02 23:51:37
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

# 网络类型-bridge
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

# 网络类型-其他
>docker network ls看到的网络

* none：容器绑定此网络后，除了回环接口外，容器没有其他网络地址
* host：容器绑定此网络后，容器直接使用宿主机的所有网络地址

# 分布式kv存储-etcd
## 简介
etcd是分布式key-value存储系统，安装etcd可用于存储docker的overlay网络信息

## 版本
* docker engine 19.03.5和 flannel v0.11版本不支持etcd v3.4.3版本，但支持3.3.x系列
* v2版本api下docker连接到etcd
```
Dec  2 14:45:40 node2 dockerd[4339]: time="2019-12-02T14:45:40.105655602Z" level=info msg="2019/12/02 14:45:40 [INFO] serf: EventMemberJoin: node2 192.1
68.2.152\n"
Dec  2 14:45:40 node2 dockerd[4339]: time="2019-12-02T14:45:40.108328635Z" level=info msg="2019/12/02 14:45:40 [INFO] serf: EventMemberJoin: node1 192.1
68.2.151\n"
```
* v3版本api下docker无法连接etcd
```
Dec  2 14:27:47 node2 dockerd[4032]: time="2019-12-02T14:27:47.113424521Z" level=warning msg="Registering as \"192.168.2.152:2375\" in discovery failed:
client: response is invalid json. The endpoint is probably not valid etcd cluster endpoint."
Dec  2 14:27:47 node2 dockerd[4032]: time="2019-12-02T14:27:47.274972145Z" level=error msg="discovery error: client: response is invalid json. The endpo
int is probably not valid etcd cluster endpoint."
```

## 安装
* 以下操作都是在双机(192.168.2.151,192.168.2.152)上运行
* 下载并解压：https://github.com/etcd-io/etcd/releases/download/v3.3.18/etcd-v3.3.18-linux-amd64.tar.gz

## 配置并启动
* 配置文件etcd.yml
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
* 启动(使用supervisor)：etcd --config-file etcd.yml

## 控制命令
- 集群成员：etcdctl member list
- 集群状态：
    + v2版本：etcdctl cluster-health
    + v3版本：etcdctl --write-out=table --endpoints=192.168.2.151:2379,192.168.2.152:2379 endpoint status/health

# 网络类型-overlay
## 简介
* 使用etcd分布式存储，docker创建的overlay网络可以实现跨主机容器互联
* 以下操作都是在双机(192.168.2.151,192.168.2.152)上运行，且需要先[安装并配置etcd](#分布式kv存储-etcd)

## docker配置调整并重启
```
{
    "registry-mirrors": ["https://2x97hcl1.mirror.aliyuncs.com"],
    "host": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"],
    "cluster-advertise": "192.168.2.151:2375",
    "cluster-store": "etcd://192.168.2.151:2379"
}
```
## 建立并使用overlay网络
* 创建网络：docker network create -d overlay demo
* 查看网络：docker network ls
* 在两台宿主机使用此网络启动：docker run -d --name test2 --network demo busybox sh -c "while true;do sleep 3600;done"
* 在两个容器内ping测试：docker exec -it test1 /bin/sh
* 在etcd中查看网络信息：etcdctl ls /docker

# 其他跨主机容器互联方式
## 静态路由
>阿里云ecs需要在vpc上添加静态路由

* 案例
    - 主机192.168.99.101  docker容器网络172.18.0.1/16
    - 主机192.168.99.102  docker容器网络172.17.0.1/16
* 操作：
    - 101主机：route add -net 172.17.0.0/16 gw 192.168.99.102
    - 102主机：route add -net 172.18.0.0/16 gw 192.168.99.101

## overlay网络第三方实现-ovs
* ovs：open vswitch
* ovs封装方式：
    - vxlan：udp报文封装
    - gre：点对点tcp报文封装

### 架构设计
|                  192.168.99.102                  |                             192.168.99.101                            |
|--------------------------------------------------|-----------------------------------------------------------------------|
| ubuntu                                           | centos7                                                               |
| ovs安装方式： apt-get install openvswitch-switch | 安装参考：http://blog.csdn.net/xinxing__8185/article/details/51900444 |
| 使用默认的docker0，网络为172.17.0.0              | 使用自定义网桥bridge0，网络为172.18.0.0                               |

### 102主机操作
```
ovs-vsctl add-br br0                   #添加名称为br0的ovs网桥
ovs-vsctl add-port br0 gre0 -- set interface gre0 type=gre options:remote_ip=192.168.99.101 #创建到另一台主机的gre tunnel
brctl addif docker0 br0              #将br0网桥添加到docker0网桥中
ip link set dev br0 up                 #启动br0
ip link set dev docker0 up    #启动docker0
ip route add 172.18.0.0/16 dev docker0        #添加到172.18.0.0/16的路由，使其通过docker0
service docker restart                #重启docker
docker start mysql-instance       #启动容器
```
### 101主机操作
```
ovs-vsctl add-br br0
ovs-vsctl add-port br0 gre0 -- set interface gre0 type=gre options:remote_ip=192.168.99.102
brctl addif bridge0 br0
ip link set dev br0 up
ip link set dev bridge0 up
ip route add 172.17.0.0/16 dev bridge0
systemctl restart docker.service
docker start  registry
```
### 持久化设置
>以centos7为例

* 建立网桥文件(/etc/sysconfig/network-scripts/ifcfg-bridge0)

```
ONBOOT=yes
BOOTPROTO=static
IPADDR=172.18.0.1
NETMASK=255.255.0.0
GATEWAY=172.18.0.0
USERCTL=no
TYPE=Bridge
DEVICE=bridge0
IPV6INIT=no
```

* 其他设置(/etc/rc.local)

```
ip route del 172.17.0.0/16 dev bridge0
brctl addif bridge0 br0
ip link set dev br0 up
```

* 保证rc.local执行：chmod +x /etc/rc.d/rc.local
