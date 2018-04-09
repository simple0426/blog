---
title: Kubernetes入门
date: 2018-04-03 10:28:44
tags: ['Kubernetes', 'k8s']
categories: ['docker']
---
# 微服务
## 服务特点
* 先天分布式【每个服务都有多个实例进行负载均衡】
* 无状态

## 无状态
><http://kyfxbl.iteye.com/blog/1831869>

* 有状态【stateful】:对单次请求的处理依赖保存的数据【比如商城购物，放入购物车、确认订单、支付等步骤需要session来实现有状态的服务】；容易实现事务；但不利于水平扩展。
* 无状态【stateless】:对单次请求的处理不依赖其他请求；容易水平扩展；为了实现事务机制，需要额外的动作【隐藏表单、sessionstorage、cookie】剥离session来实现服务从有状态到无状态转变。

# 简介
kubernetes是google开源的容器集群管理系统。它构建于docker技术之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等提供一套功能，本质上可看做基于容器技术的pass平台

# 基本概念
## namespace
用于资源隔离，不同ns中的资源不能相互访问，是多租户下常见的解决方案

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

## pod
* 一组容器的集合体
* 是k8s中最小任务调度单元
* 一个pod里的所有容器共享资源【volumes/网络】

## replication controller
* pod的复制抽象，用于解决pod的扩容/缩容问题
* 控制某个pod的当前实例个数【在所有minion node中创建指定数量的实例】
* 包含创建pod实例的模板

## service
* pod的路由代理抽象，用于解决pod之间的服务发现和负载均衡。
* service的引入旨在保证pod的动态变化【比如切换机器，缩容过程中被终止了等】对访问端透明，访问端只需要知道service地址，由service来提供代理
* 每个Service自动分配一个clusterip和DNS名，用于集群内各服务之间的调用，外部不可访问

## deployments
* 一种更加简单的更新RC和pod的机制
* 通过在deploy中描述你所希望的集群状态，在一个可控的速度下逐步更新成你所期望的集群状态
* Deployment主要职责同样是为了保证pod的数量和健康，90%的功能与RC完全一样，可以看成是新一代的RC
    - 时间和状态查看
    - 版本记录和回滚操作
    - 随时暂停和启动
    - 多种升级方案
        + Recreate----删除所有已存在的pod,重新创建新的
        + RollingUpdate----滚动升级，逐步替换的策略

# 核心组件
## master组件
* etcd：分布式配置存储服务，保存集群状态，类似zookeeper
* apiserver：是kubernetes系统的入口，封装了对核心对象的增删改查操作，以restful接口的形式提供给外部和内部使用。它维护的rest对象可以持久化到etcd中
* scheduler：负责集群资源调度，根据预定的调度策略将pod调度到相应的机器上
* controller-manager：负责维护集群的状态，比如故障检测、自动扩展、滚动更新等
    - 节点（Node）控制器
    - endpoint-controller：定期关联service和pod
    - replication-controller：负责关联replication controller和pod
    - Service Account和Token控制器：为新的Namespace 创建默认帐户访问API Token

## minion组件
* kublet：
    * 负责维护容器的生命周期，比如启动停止等；
    * 也负责Volume(CVI)和网络的管理(CNI)。
    * 它会定期从etcd中获取本机的pod，并根据pod信息启动和停止相应的容器。同时接受apiserver的http请求，汇报pod运行状态；
* kube-proxy：
    - 负责为service提供cluster内部的服务发现和负载均衡
    - 负责为pod提供代理，他会定期从etcd获取所有的service，并根据service信息创建代理。
    - 当某个客户访问pod时需要访问其他pod时，访问请求会经过本机的proxy做转发
* container runtime
    - 容器运行环境：负责镜像管理以及pod和容器的真正运行(CRI)，比如docker

## 附件组件
* kube-dns 负责为整个集群提供DNS服务
* kubenetes-dashboard 提供GUI
