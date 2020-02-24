---
title: kubernetes入门
tags:
  - kubernetes
  - k8s
categories:
  - kubernetes
date: 2018-04-03 10:28:44
---

# 核心功能
* 服务发现、负载均衡
* 容器运行、灾难恢复
* 应用的发布、回滚，配置管理
* 扩容、缩容等弹性伸缩
* 批量执行任务

# 架构和功能组件
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/k8s%E6%9E%B6%E6%9E%84.png)
## master节点
>管理和数据资源入口

* etcd：分布式配置存储服务，保存集群状态，类似zookeeper
* apiserver：是kubernetes系统的入口，封装了对核心对象的增删改查操作，以restful接口的形式提供给外部和内部使用。它维护的rest对象可以持久化到etcd中
    - 可以水平扩展进行部署
* scheduler：负责集群资源调度，根据预定的调度策略将pod调度到相应的机器上
    - 可以热备，但只能有一个active
* controller-manager：负责维护集群的状态，比如故障检测、自动扩展、滚动更新等
    - 可以热备，但只能有一个active
    - 节点（Node）控制器
    - endpoint-controller：定期关联service和pod
    - replication-controller：负责关联replication controller和pod
    - Service Account和Token控制器：为新的Namespace 创建默认帐户访问API Token

## node节点
>真正运行业务负载的节点

* kublet：
    * 负责维护容器的生命周期，比如启动停止等；
    * 通过存储插件、网络插件，管理Volume(CVI)和网络(CNI)。
    * 它会定期从etcd中获取本机的pod，并根据pod信息启动和停止相应的容器。同时接受apiserver的http请求，汇报pod运行状态；
* kube-proxy：
    - 负责为service提供cluster内部的服务发现和负载均衡
    - 负责为pod提供代理，他会定期从etcd获取所有的service，并根据service信息创建代理。
    - 当某个客户访问pod时需要访问其他pod时，访问请求会经过本机的proxy做转发
* container runtime
    - 容器运行环境：负责镜像管理以及pod和容器的真正运行(CRI)，比如docker

## 组件交互范例
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/k8s-module-interact.jpg)

* 用户通过UI或CLI提交一个pod给API Server进行部署
* API Server将信息写入etcd存储
* Scheduler通过API Server的watch或notification机制获取这个信息
* Scheduler通过资源使用情况进行调度决策(pod在哪个节点部署)，并把决策信息返回给API server
* API server将决策结果写入etcd
* API server通知相应的节点进行pod部署
* 相应节点的kubelet得到通知后，调用container runtime启动容器、调度存储插件配置存储、调度网络插件配置网络

# 基本概念
## namespace
用于集群内部的逻辑隔离，每个resource都属于一个namespace，一个namespace内的资源命名具有唯一性

## node
Pod真正运行的主机，可以是物理机，可以是虚拟机。为了管理Pod，每个Node节点至少要运行container runtime、kubelet和kube-proxy服务。

## resource
* k8s中几乎所有的重要概念都是resource
* 它处于某个namespace中
* 可以持久化到etcd中
* 资源可以限定使用配额

## label
* 用于区分pod、service、replication controller的key-value对
* 贴在resource上，用来过滤和选择资源
* service通过label selector选择一组pod来提供服务
* rc通过label管理通过pod模板创建的一组pod

# 资源-REST对象
kubernetes以RESTful的形式开放 接口，用户可操作的REST对象如下
## Pod
* 具有相同namespace的一组容器的集合体
* 是k8s中最小任务调度单元
* 一个pod里的所有容器共享资源【volumes/网络】

## Replication controller
* pod的复制抽象，用于解决pod的扩容/缩容问题
* 控制某个pod的当前实例个数【在所有node中创建指定数量的实例】
* 包含创建pod实例的模板

## Deployment
* 一种更加简单的更新RC和pod的机制
* 通过在deploy中描述你所希望的集群状态，在一个可控的速度下逐步更新成你所期望的集群状态
* Deployment主要职责同样是为了保证pod的数量和健康，90%的功能与RC完全一样，可以看成是新一代的RC
    - 时间和状态查看
    - 版本记录和回滚操作
    - 随时暂停和启动
    - 多种升级方案
        + Recreate----删除所有已存在的pod,重新创建新的
        + RollingUpdate----滚动升级，逐步替换的策略

## Service
* 可以提供一个或多个pod实例的稳定访问地址【pod的动态变化，比如切换机器，缩容过程中被终止了等，对访问端透明】
* 实现方式
    - ClusterIP
    - NodePort
    - LoadBalancer

# 资源-yaml文件
```
apiVersion: v1 #接口版本
kind: Pod #资源类型
metadata: #资源的概括定义，比如：名称、标签
  name: nginx
  lables: #资源的标签，可以被selector使用
    name: nginx
spec: #资源的详细定义
  containers:
  - name: nginx
     image: nginx
     ports:
     - containerPort: 80
```
# 简单使用-minikube
## 安装
* [下载安装minikube](https://github.com/kubernetes/minikube)
    - minikube是类似于vagrant的工具，需要借助本地虚拟化的支持（hyper-v、virtualbox）
* [下载安装kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-windows)
* 启动命令：`minikube start --image-mirror-country=cn --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --registry-mirror=https://2x97hcl1.mirror.aliyuncs.com`
    - start其他配置选项
        + --cpus=2：为minikube虚拟机配置的cpu数量
        + --disk-size='20000mb'：为minikube虚拟机配置的磁盘大小
        + --memory='2000mb'：为minikube虚拟机分配的内存
        + --kubernetes-version='v1.16.2'：minikube使用的kubernetes版本
    - 因为需要操作网卡，所以需要root权限（windows下使用管理员身份启动）
* 停止命令：minikube stop
* 浏览器查看集群信息：minikube dashboard
* 使用kubectl查看集群信息：kubectl config view、kubectl cluster-info

## 使用
* 部署：kubectl apply -f https://k8s.io/examples/application/deployment.yaml
* 升级镜像：kubectl apply -f https://k8s.io/examples/application/deployment-update.yaml
* 扩容：kubectl apply -f https://k8s.io/examples/application/deployment-scale.yaml

## kubectl操作
* kubect get node：查看节点状态
* kubectl get --watch deployments：持续查看deployments信息
* kubectl describe deployment：显示deployments详情
* kubectl delete deployment nginx-deployment：删除deployment
