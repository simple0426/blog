---
title: 容器技术docker
tags:
categories:
---
# 虚拟化技术
* OpenStack：整体数据中心的管理方案
* kvm：基于内核实现的虚拟化解决方案，一般用于多租户计算资源管理；KVM整体效率最高
* vmware：和kvm一样，是基于内核的虚拟化解决方案；功能全面
* virtualbox：与vmware类似，但比vmware轻量、运行效率也比vmware高；oracle开源产品，中文用户多。
* docker/coreos：当前容器技术标准
    - 与传统虚拟化的区别：容器是在操作系统层面实现虚拟化，直接复用本地操作系统内核，它的技术基础是linux容器技术（LXC，即cgroups）；传统虚拟化方式则基于硬件层面实现虚拟化。
    - 由于目前docker不支持热迁移，可以将docker容器放在KVM虚拟机里，通过虚拟机的迁移完成容器的迁移

# docker基本概念
* 容器：视图隔离、资源可限制、独立文件系统的进程集合
    - docker利用容器来运行应用
    - 容器是从镜像创建的运行实例，它可以被开始、停止、删除。
    - 每个容器都是相互隔离的
    - 可以把容器看做是一个简易的linux环境和运行在其中的应用程序
    - 镜像是只读的，容器在启动时会创建一个可写层作为多层文件系统的最上层
* 镜像：容器运行所需的所有文件集合，它是一个只读模板，用来创建容器
* 镜像仓库：存放镜像文件的场所

## 分层文件系统
1. 镜像拆分成多个模块，可以并行下载，提高分发效率
2. 镜像数据是共享的，每次下载或构建镜像时，可以复用已存在的部分数据，此时，只需存储本地不存在的数据。

# docker产品分类
* moby：是继承了原先的docker的项目，是社区维护的的开源项目，谁都可以在moby的基础打造自己的容器产品
* docker-ce是docker公司维护的开源项目，是一个基于moby项目的免费的容器产品
* docker-ee是docker公司维护的闭源产品，是docker公司的商业产品。

# docker工具集
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

# linux安装
* 阿里云安装参考：https://yq.aliyun.com/articles/110806
* centos安装注意
    - 关闭firewall：systemctl stop firewalld
    - 安装并开启iptables：systemctl start|enable iptables

# 容器
## 容器核心技术
* 资源视图隔离：namespace
* 资源使用限制(cpu、内存)：cgroups
* 独立文件系统：chroot（linux下的解决方案）

## 容器运行
* 下载镜像(本地不存在时，自动从远程仓库下载)：docker pull centos:7
* 查看镜像列表：docker images
* 选择镜像并运行（run）
    - 范例1：docker run -idt -m 100m --memory-swap=100m ubuntu /bin/bash

## run命令
### 常用参数
>docker run -idt -v /home/Logs/:/root/Logs:ro -m 100m --memory-swap=100m --cpus 0.5 -p 10086:22 sshd

* -t：分配一个伪终端tty
* -i：保持容器的标准输入一直打开
* -d：让容器以后台方式运行
* -v /home/Logs/:/root/Logs:ro（可多次使用）：映射宿主机/home/Logs到容器/root/Logs并设置为只读
* -m 100m --memory-swap=100m：设置容器内存为100m、内存和swap总和也是100m
* --cpus 0.5：设置容器可以使用的cpu百分比（最大值为cpu的核心数，最小值为0.1，可以是小数）
* -p 10086:22（可多次使用）：映射宿主机端口10086到容器端口22

### 其他参数
* --cidfile：把容器id写入一个文件
* --name：给运行的容器绑定一个名称
* --rm：容器存在就删除【运行一次性容器，运行后立即删除】
* -h：配置容器主机名
* --dns：配置容器dns
* -P：将主机的49000~49900的端口随机映射到内部容器的开放端口
* --link name:alias：链接容器【容器名称：别名】，这种链接关系可以在容器hosts文件看到

### publish与expose
* publish：将容器端口映射到宿主机【命令行参数-p或-P】
* expose：曝露端口用于容器间的访问【Dockerfile文件中的关键字EXPOSE】

## 其他管理命令
* start、stop：启动、停止一个容器
* create、rm：创建、删除容器
* pause/unpause：挂起、恢复容器内的所有进程
* port：端口映射查看
* logs [-f]：日志查看，-f相当于tailf效果
* exec：在容器中执行命令：docker exec -it 5c8f798e206e /bin/bash
* ps [-a]：查看运行中的容器，-a查看所有存在的容器
* inspect：查看容器或镜像信息
* stats：查看容器资源使用情况
* top：查看进程列表

## volume管理
### 使用原因
* 原因：docker存储引擎device mapper、aufs等实现的copy on write机制，在高频写操作下性能不高
* 优势：使用volume，可以直接写磁盘；可以实现目录持久化；可以实现宿主机与容器、容器之间的数据共享

### 使用方式
* 映射宿主机目录到容器：◦docker run -idt --name test1 -v ~/nginx:/webapps ubuntu /bin/bash
* 数据卷使用
    - 建立数据卷：docker volume create hello
    - 其他容器使用数据卷：docker run -d -v hello:/world busybox ls /world
* 卷容器使用
    - 建立卷容器（容器无需一直运行）：◦docker run -d -v ~/dbdata:/dbdata --name dbdata ubuntu echo data-only
    - 其他容器挂载使用：◦docker run -idt --volumes-from dbdata --name db2 ubuntu /bin/bash
