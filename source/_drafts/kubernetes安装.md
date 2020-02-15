---
title: kubernetes安装
tags:
categories:
---
# 安装方式
* kubeadm：多节点集群安装：https://github.com/kubernetes/kubeadm
* 源码下载地址：
    - https://storage.googleapis.com/kubernetes-release/release/v1.16.2/kubernetes-server-linux-amd64.tar.gz
    - https://dl.k8s.io/v1.16.2/kubernetes-server-linux-amd64.tar.gz

# kubectl命令
## 多集群管理
* 合并多个集群的config文件（clusters、contexts、users）
* 操作命令
    - 查看config配置：kubectl config view
    - 查看当前管理的集群：
    - 切换管理的集群：kubectl config use-context minikube
