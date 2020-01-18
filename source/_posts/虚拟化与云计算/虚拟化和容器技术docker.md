---
title: 虚拟化和容器技术docker
tags:
  - 虚拟化
  - 容器
  - docker
categories:
  - 虚拟化
date: 2019-11-19 23:24:32
---

# 虚拟化
## 产生背景
* 部署非常慢：采购服务器、安装操作系统、安装软件环境
* 资源浪费：硬件需要预采购，成本很高；但是会面临cpu、内存等资源使用不足
* 软件难于迁移和扩展
* 软件被限定只能运行在特定硬件上

## 技术优势
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/virtual-tech.jpg)

* 资源池：一个物理机的资源可以分配到不同虚拟机里
* 容易扩展：加物理机或加虚拟机
* 容易云化：比如AWS、阿里云等

## 虚拟化产品
* openstack：数据中心级别的虚拟化管理方案
* kvm：基于内核实现的虚拟化解决方案，一般用于多租户计算资源管理；KVM整体效率最高
* vmware：和kvm一样，是基于内核的虚拟化解决方案；功能全面
* virtualbox：与vmware类似，但比vmware轻量、运行效率也比vmware高；oracle开源产品，中文用户多。

## 虚拟化局限性
每一个虚拟机都是一个完整的操作系统，操作系统本身需要消耗资源

# 容器
## 容器出现
通过标准化“应用运行环境”，沟通开发(dev)和运维(ops)流程

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/devops-demo.jpg)
## 容器特点
* 对软件和其依赖的标准化打包
* 应用之间相互隔离
* 共享同一个os内核
* 可以运行在多个主流操作系统上

## 容器与虚拟机
* 区别：容器是在操作系统层面实现虚拟化，直接复用本地操作系统内核，它的技术基础是linux容器技术（LXC，即cgroups）；传统虚拟化方式则基于硬件层面实现虚拟化。

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/containerVSvm.jpg)

* 融合：由于目前docker不支持热迁移，可以将docker容器放在KVM虚拟机里，通过虚拟机的迁移完成容器的迁移

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/container%2BVM.jpg)

# docker
## 基本概念
* 容器：视图隔离、资源可限制、独立文件系统的进程集合
    - 可以把容器看做是一个简易的linux环境和运行在其中的应用程序
    - docker利用容器来运行应用
    - 容器是从镜像创建的运行实例，它可以被开始、停止、删除。
    - 每个容器都是相互隔离的
* 镜像：容器运行所需的所有文件集合
    + 它是一个只读模板，用来创建容器
    + 容器在启动时会创建一个可写层作为多层文件系统的最上层
* 镜像仓库：存放镜像文件的场所

## 核心技术
* 视图隔离(namespace)：pid、net、ipc、mnt、uts
* 资源限制(cgroups)：cpu、内存
* 联合文件系统(union file system)：容器和镜像的分层，即分层文件系统
    + 镜像拆分成多个模块，可以并行下载，提高分发效率
    + 镜像数据是共享的，每次下载或构建镜像时，可以复用已存在的部分数据，此时，只需存储本地不存在的数据。
    + 容器在启动时会创建一个可写层作为多层文件系统的最上层

## 相关产品
* moby：是继承了原先的docker的项目，是社区维护的的开源项目，谁都可以在moby的基础打造自己的容器产品
* docker-ce是docker公司维护的开源项目，是一个基于moby项目的免费的容器产品
* docker-ee是docker公司维护的闭源产品，是docker公司的商业产品。

## 工具集
* [compose](https://github.com/docker/compose)：简单的多容器部署工具，通过简单的命令和配置，管理服务于某个应用的多个容器
* [machine](https://github.com/docker/machine)：将docker安装在指定主机，比如虚拟机、本地、远端云主机
* [kitematic](https://github.com/docker/kitematic)：桌面版的docker客户端
* [toolbox](https://github.com/docker/toolbox)：将docker安装在mac和windows下的传统方式【官方已有新的解决方案】，安装包含如下
    - docker machine
    - docker engine【docker command】
    - docker compose
    - kitematic
    - shell command-line
    - Oracle VirtualBox
* swarm：容器编排工具

## 软件安装
* 阿里云安装参考：https://yq.aliyun.com/articles/110806

## engine配置
>/etc/docker/daemon.json

```
{
  "registry-mirrors": ["https://2x97hcl1.mirror.aliyuncs.com"],
  "insecure-registries": ["reg.cnabke.com"],
  "data-root": "/data/docker-data",
  "bridge": "bridge0",
  "storage-driver": "devicemapper",
  "debug": true,
  "storage-opts" : [
    "dm.basesize=5G"
  ]
}
```

* registry-mirrors：公有registry或加速器地址
* insecure-registries：私有registry地址
* data-root：容器、镜像等资源的存储位置
* bridge：设置docker容器的默认网络地址
  - 绑定网络地址(docker engine启动参数)：--bip 172.18.0.1
  - 绑定网桥(docker engine启动参数)：-b, --bridge bridage_name
* storage-driver：容器存储驱动
* storage-opts：存储驱动的选项
    - dm.basesize=5G：限制容器磁盘的最大容量

## 图形化
* 使用软件：portainer
* 启动：docker run -d -p 9000:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock --name portainer portainer/portainer
* web访问：http://ip:9000

## 常见问题
### 问题1
* 现象：通过路由实现跨主机docker容器互联，如果ping对端容器出现：Dest Unreachable, Unknown Code: 10或Destination Host Prohibited
* 解决：则是由于对端宿主机开启防火墙或iptables原因导致

### 问题2
* 现象：docker容器网络不通
* 解决
    - 确定net.ipv4.conf.all.forwarding=1
    - 重启docker服务【单独重启即可】

### 问题3
* 现象：禁止普通用户使用docker命令
```
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.40/images/json: dial unix /var/run/docker.sock: connect: permission denied
```
* 解决：
    + 将当前用户加入docker组中：gpasswd -a python docker
    + 重启docker进程：systemctl restart docker
    + 重新登录服务器终端
