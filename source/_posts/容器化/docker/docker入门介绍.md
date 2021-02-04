---
title: docker入门介绍
tags:
  - 虚拟化
  - 容器
categories:
  - docker
date: 2019-11-19 23:24:32
---

# docker引入
## 传统IT面临问题
* 部署流程繁琐(采购服务器、安装软件环境)、软件环境臃肿、难于扩展(不利于迅速应对业务高峰对资源需求)
* 环境不一致：开发、测试、生产环境不一致
* 资源浪费：硬件需要预采购，成本很高；同时面临cpu、内存等资源使用不足

## 容器优势
* 更快且保持一致性的交付应用程序【保持多阶段环境的一致性】
* 响应式部署和扩展【可以在物理机、虚拟机、云等多种环境部署，并可以根据负载弹性伸缩】
* 在同一个硬件上运行更多的负载

## 容器化devops
通过标准化“应用运行环境”，沟通开发(dev)和运维(ops)流程

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/devops-demo.jpg)

## 容器与虚拟机
* 区别：容器是在操作系统层面实现虚拟化，直接复用本地操作系统内核，它的技术基础是linux容器技术（LXC，即cgroups）；传统虚拟化方式则基于硬件层面实现虚拟化。

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/containerVSvm.jpg)

* 融合：由于目前docker不支持热迁移，可以将docker容器放在KVM虚拟机里，通过虚拟机的迁移完成容器的迁移

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/container%2BVM.jpg)

# [docker介绍](https://docs.docker.com/get-started/overview/#docker-architecture)
## docker架构
* docker daemon(dockerd)：监听API请求、管理docker对象（镜像、容器、网络、卷）
* docker client：用户管理docker的入口
* docker registry：镜像仓库

![](https://docs.docker.com/engine/images/architecture.svg)

## docker对象
* 容器：视图隔离、资源可限制、独立文件系统的进程集合
    - 可以把容器看做是一个简易的linux环境和运行在其中的应用程序
    - docker利用容器来运行应用
    - 容器是从镜像创建的运行实例，它可以被开始、停止、删除。
    - 每个容器都是相互隔离的
* 镜像：容器运行所需的所有文件集合
    + 它是一个只读模板，用来创建容器
    + 容器在启动时会创建一个可写层作为多层文件系统的最上层

## 核心技术
* 视图隔离(namespace)：pid、net、ipc、mnt、uts
* 资源限制(cgroups)：cpu、内存
* 联合文件系统(union file system)：容器和镜像的分层，即分层文件系统
    + 镜像拆分成多个模块，可以并行下载，提高分发效率
    + 镜像数据是共享的，每次下载或构建镜像时，可以复用已存在的部分数据，此时，只需存储本地不存在的数据。
    + 容器在启动时会创建一个可写层作为多层文件系统的最上层
* 容器格式(container format)：docker engine将namespace、cgroups、UnionFS组合到容器格式的包装器中，默认的容器格式为libcontainer

## docker产品
* [docker engine](https://docs.docker.com/engine/)：docker服务端引擎
* [docker compose](https://docs.docker.com/compose/)：简单的多容器部署工具，通过简单的命令和配置，管理服务于某个应用的多个容器
* [docker desktop](https://docs.docker.com/desktop/)：适用于Mac或Windows环境的易于安装的应用程序，可让您构建和共享容器化的应用程序和微服务；
* [docker hub](https://hub.docker.com/repositories)：公共镜像仓库

## 过时的工具集
* [machine](https://docs.docker.com/machine/overview/)：将docker安装在指定主机，比如虚拟机、本地、远端云主机
* [kitematic](https://docs.docker.com/kitematic/userguide/)：桌面版的docker客户端
* [toolbox](https://docs.docker.com/toolbox/overview/)：将docker安装在mac和windows下的传统方式，安装包含如下
    - docker machine
    - docker engine【docker command】
    - docker compose
    - kitematic
    - shell command-line
    - Oracle VirtualBox
* swarm：docker默认支持的容器编排工具

# docker安装与配置
## 软件安装-CentOS
* [官方(较慢)](https://docs.docker.com/engine/install/centos/#install-using-the-repository)

  ```
   sudo yum install -y yum-utils
   sudo yum-config-manager --add-repo \
      https://download.docker.com/linux/centos/docker-ce.repo
   sudo yum install docker-ce docker-ce-cli containerd.io
   sudo systemctl start docker
  ```
  
* [阿里云镜像](https://developer.aliyun.com/mirror/docker-ce)

  ```
  # step 1: 安装必要的一些系统工具
  sudo yum install -y yum-utils device-mapper-persistent-data lvm2
  # Step 2: 添加软件源信息
  sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  # Step 3
  sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
  # Step 4: 更新并安装Docker-CE
  sudo yum makecache fast
  sudo yum -y install docker-ce
  # Step 4: 开启Docker服务
  sudo service docker start
  ```

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

# 运维管理
## 图形化
* 使用软件：portainer
* 启动：`docker run -d -p 9000:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock --name portainer portainer/portainer`
* web访问：http://ip:9000

## 数据采集
* 使用软件：[cadvisor][1]
* 服务启动
```
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  --device=/dev/kmsg \
  google/cadvisor:latest
```

## 监控报警
* 使用软件：[prometheus][3] 、[github地址](https://github.com/prometheus)
* 服务启动：`docker run --name prometheus -d -p 9090:9090 prom/prometheus`
* 添加cadvisor数据【重启prometheus】：/etc/prometheus/prometheus.yml
```
  - job_name: 'cadvisor'                 
                                
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
                                 
    static_configs:     
    - targets: ['172.17.8.51:8080','172.17.8.52:8080']
```

## 度量分析和可视化系统
* 使用软件：[grafana][2]
* 服务启动：`docker run -d -p 3000:3000 --name grafana grafana/grafana`
* 添加数据源-prometheus
* 添加docker监控模板【ID：193】

[1]: https://github.com/google/cadvisor
[2]: https://grafana.com/grafana/
[3]: https://prometheus.io/
