---
title: kubernetes-入门基础
tags:
  - kubernetes
  - k8s
categories:
  - kubernetes
date: 2018-04-03 10:28:44
---

# 核心功能
* 自动装箱：构建于容器技术之上，基于资源依赖和其他约束性自动完成程序部署；通过调度机制，提升节点资源利用率
* 自我修复：支持容器故障后自动重启，节点故障后重新调度容器
* 水平扩展：支持通过命令、UI或HPA方式完成扩容、缩容等弹性伸缩
* 服务发现、负载均衡：通过使用DNS插件为系统内置服务发现功能，它为每个service配置DNS名称；而service通过iptables或ipvs内建了负载均衡机制
* 应用的发布、回滚：支持自动滚动更新，保证服务可用性；在出现故障时，支持回滚操作
* 秘钥和配置管理：通过ConfigMap实现配置管理，使用secret保证数据安全性
* 存储编排：支持pod对象按需挂载不同类型的存储系统，例如：本地存储(hostpath)、云存储(oss)、网络存储(glusterfs)
* 批量执行任务

# 概念术语
## name和namespace
* name是资源对象的标识，作用域为namespace
* namespace：用于限定资源的作用域
  - 用于集群内部的逻辑隔离（资源对象名称隔离）
  - 不能隔离不同名称空间的pod间通信，这是网络策略(network policy)的功能
  - 默认名称空间为default

## label和selector
* label：用于资源分类的key-value对，用来过滤和选择资源
* selector：根据标签(label)选择和过滤资源的一种机制

## Pod
* 具有相同namespace的一组容器的集合体
* 是k8s中最小任务调度单元
* 一个pod里的所有容器共享资源【volumes/网络】

## pod控制器
* pod控制器是工作负载类控制器，用于部署和管理pod对象
* 工作负载类控制器包括：ReplicationController、ReplicaSet、Deployment、Statefulset、job等

## Service
* 建立在pod对象之上的资源抽象，通过标签选择器选定一组pod，并为这组pod对象定义一个统一且固定的访问入口【集群ip或DNS名称】
* 实现方式
    - ClusterIP
    - NodePort
    - LoadBalancer
    - ExternalName

# 架构和功能组件
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/k8s%E6%9E%B6%E6%9E%84.png)
## master组件
* etcd：分布式配置存储服务，保存资源的定义和状态；它不属于kubernetes集群自身
* apiserver：是kubernetes系统的入口，封装了对资源对象的增删改查操作，以restful接口的形式提供给外部和内部使用。它维护的rest对象持久化到etcd中
* scheduler：负责集群资源调度，根据预定的调度策略将pod调度到相应的机器上
* controller-manager：确保各资源的当前状态(status)可以匹配用户期望的状态(spec)；实现此功能的操作有：故障检测、自动扩展、滚动更新等；实现此功能的控制器有：各类pod控制器，endpoint控制器、ServiceAccount控制器、node控制器等

## node组件
* kublet：
    * 从apiserver接收关于pod对象的配置信息，并确保他们处于期望的状态
    * 在APIServer注册当前节点，并汇报当前节点的资源使用情况
    * 通过存储插件、网络插件，管理Volume(CVI)和网络(CNI)。
* kube-proxy：按需为service对象生产iptables或ipvs规则，从而捕获访问当前service的流量将其转发到正确的后端pod对象
* container runtime(容器运行环境)：负责镜像管理以及容器的真正运行(CRI)，比如docker

## 核心附件
* DNS组件(CoreDNS):为集群提供服务注册和服务发现的动态名称解析服务
* UI组件(Kubernetes-dashboard):为集群提供web页面管理功能
* 监控组件：收集容器和节点的状态信息
    + 早期使用heapster
    + 现在使用metrics server+prometheus
* IngressController：在service之上，为集群提供7层代理；实现项目:nginx、traefik

## 组件交互范例
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/k8s-module-interact.jpg)

* 用户通过UI或CLI提交一个pod给API Server进行部署
* API Server将信息写入etcd存储
* Scheduler通过API Server的watch或notification机制获取这个信息
* Scheduler通过资源使用情况进行调度决策(pod在哪个节点部署)，并把决策信息返回给API server
* API server将决策结果写入etcd
* API server通知相应的节点进行pod部署
* 相应节点的kubelet得到通知后，调用container runtime启动容器、调度存储插件配置存储、调度网络插件配置网络

# 简单使用-[minikube](https://minikube.sigs.k8s.io/)
## 软件安装
* [下载安装minikube](https://github.com/kubernetes/minikube)
    - minikube是类似于vagrant的工具，需要借助本地虚拟化的支持（hyper-v、virtualbox）
* [下载安装kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-windows)

## 注意事项
* minikube命令和minikube项目存储位置(MINIKUBE_HOME，也即.minikube的上级目录)应该在同一分区上(例如都在c盘或d盘),否则minikube启动会找不到minikube的iso文件
* 默认项目存储位置(MINIKUBE_HOME)在用户家目录(如C:\Users\simple)，可以设置MINIKUBE_HOME变量改变
* 项目启动时会下载minikube的iso文件、kubectl、kubelet、kubeadm等组件，它们都是从[境外](https://storage.googleapis.com)下载的，为了快速下载，应该在23点之后进行首次下载并缓存

## 集群管理
* 启动命令：`minikube start --image-mirror-country=cn --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --registry-mirror=https://2x97hcl1.mirror.aliyuncs.com --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.7.3.iso`
    - start其他配置选项
        + --cpus=2：为minikube虚拟机配置的cpu数量
        + --disk-size=20000mb：为minikube虚拟机配置的磁盘大小
        + --memory=2000mb'：为minikube虚拟机分配的内存
        + --kubernetes-version=v1.16.2：minikube使用的kubernetes版本
* 停止命令：minikube stop
* web界面使用：minikube dashboard
* 集群信息查看：kubectl config view、kubectl cluster-info

## 应用管理
* 部署：kubectl apply -f https://k8s.io/examples/application/deployment.yaml
* 升级镜像：kubectl apply -f https://k8s.io/examples/application/deployment-update.yaml
* 扩容：kubectl apply -f https://k8s.io/examples/application/deployment-scale.yaml

## 集群运维
* kubect get node：查看节点状态
* kubectl get --watch deployments：持续查看deployments信息
* kubectl describe deployment：显示deployments详情
* kubectl delete deployment nginx-deployment：删除deployment
