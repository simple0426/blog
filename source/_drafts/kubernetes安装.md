---
title: kubernetes安装
tags:
categories:
---
# 主要安装方式
* minikube：单节点集群安装：https://github.com/kubernetes/minikube
    - minikube是类似于vagrant的工具，需要借助本地虚拟化的支持（hyper-v、virtualbox）
* kubeadm：多节点集群安装：https://github.com/kubernetes/kubeadm

# 其他安装方式
* 下载地址：
    - https://storage.googleapis.com/kubernetes-release/release/v1.16.2/kubernetes-server-linux-amd64.tar.gz
    - https://dl.k8s.io/v1.16.2/kubernetes-server-linux-amd64.tar.gz
* 阿里云kubernetes服务
* kubectl windows安装：https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-windows

# minikube安装
* 启动命令：minikube start --image-mirror-country=cn --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --registry-mirror=https://registry.docker-cn.com
    - 因为需要操作网卡，所以需要root权限（windows下使用管理员身份启动）
* 停止命令：minikube stop
* 浏览器查看集群信息：minikube dashboard
* 使用kubectl查看集群信息：kubectl config view、kubectl cluster-info

# kubectl命令
## 多集群管理
* 合并多个集群的config文件（clusters、contexts、users）
* 操作命令
    - 查看config配置：kubectl config view
    - 查看当前管理的集群：
    - 切换管理的集群：kubectl config use-context minikube
