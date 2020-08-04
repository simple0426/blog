---
title: kubernetes-容器技术落地方案
tags:
categories:
---

# 容器平台分层架构

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/containers-scheme.jpg)

* 基础设施层-IaaS(openstack/公有云)：服务器、网络、存储、数据库
* 容器引擎层：docker、harbor(镜像仓库)
* 容器编排层：kubernetes
* 访问和工具层-PaaS：web控制台、CICD、监控、日志

# 技术方案选择

## 服务器

* 服务器类型：物理机、虚拟机
* 节点管理【ssd、GPU】
  * 定义节点标签；pod使用亲和性调度
  * 定义节点污点；pod使用容忍度调度

## 存储

* 主流方案-ceph
  * 对象存储：类似oss，程序直接调用
  * 文件存储：类似nfs，直接挂载使用
  * 块存储：类似硬盘、分区，需要格式化为特定文件系统后才可以使用
* 其他方案：glusterfs、fastdfs

## 网络

* 方案
  * calico：纯三层网络方案
  * flannel
* calico
  * 支持k8s的网络策略(ACL)
  * 支持k8s集群与外部通信
    * 实现微服务注册中心(consul/zookeeper)中集群内外不同服务的互通
    * 代替kube-proxy，使用自有的负载均衡(如nginx)

## 镜像仓库

* 主流方案-harbor：支持权限管理、日志审计等功能
* 其他方案
  * nexus
  - jfrog
  - Dockerdistribution

## 监控

主流方案：prometheus+grafana

## 日志

elk

## CICD流程

* 主流：jenkins
* 其他方案：gitlab ci

## 其他注意事项

* 命名空间：资源隔离
* 权限控制：RBAC、网络策略
* 降低使用难度：操作简化、web界面管理等

# 资源要求和限制

## 硬件要求
| 环境     | 节点类型    | 配置        |
| -------- | ----------- | ----------- |
| 实验环境 | master/node | 2C/2G+      |
| 测试环境 | master      | 2C/4G/40G   |
| -        | node        | 4C/8G/40G   |
| 生产环境 | master      | 4C/8G/100G  |
| -        | node        | 8C/64G/500G |

## 使用限额
```
在 v1.18 版本中， Kubernetes 支持的最大节点数为 5000。更具体地说，我们支持满足以下所有条件的配置：
节点数不超过 5000
Pod 总数不超过 150000
容器总数不超过 300000
每个节点的 pod 数量不超过 100
```

# k8s集群部署

* [手动二进制]([https://simple0426.gitee.io/2020/07/12/%E5%AE%B9%E5%99%A8%E5%8C%96/kubernetes/kubernetes-%E4%BA%8C%E8%BF%9B%E5%88%B6%E6%96%B9%E5%BC%8F%E9%83%A8%E7%BD%B2/](https://simple0426.gitee.io/2020/07/12/容器化/kubernetes/kubernetes-二进制方式部署/)) ：手动分组件一步一步搭建集群；可用于学习ks8集群运行细节、也可用于生产环境
* [rancher](https://www.rancher.cn/)：企业级开源kubernetes管理平台
* 使用自动化工具搭建k8s集群
  * [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/)：官方维护，一键部署k8s集群；一般用于测试环境和学习
  * ansible手动：使用ansible编写自动化部署
    * 阿良范例：https://github.com/lizhenliang/ansible-install-k8s
  * kubespray：官方维护，基于ansible编写的部署程序
    * github：https://github.com/kubernetes-sigs/kubespray
    * web：https://kubernetes.io/docs/setup/production-environment/tools/kubespray/

