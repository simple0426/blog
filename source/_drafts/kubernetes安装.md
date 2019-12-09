---
title: kubernetes安装
tags:
categories:
---
# 安装方式
* minikube：单节点集群安装：https://github.com/kubernetes/minikube
    - minikube是类似于vagrant的工具，需要借助本地虚拟化的支持（hyper-v、virtualbox）
* kubeadm：多节点集群安装：https://github.com/kubernetes/kubeadm

## 其他安装
* 阿里云kubernetes服务
* kubectl windows安装：https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-windows

# minikube安装
* 启动命令：minikube start --image-mirror-country=cn --registry-mirror=https://fer7fge6.mirror.aliyuncs.com
* 停止命令：minikube stop
* 浏览器查看集群信息：minikube dashboard
* 使用kubectl查看集群信息：kubectl config view、kubectl cluster-info
