---
title: kubernetes-二进制部署
tags:
categories:
---
# 二进制下载
1. 登录https://github.com/kubernetes/kubernetes/releases
2. 从github下载在线安装包：https://github.com/kubernetes/kubernetes/releases/download/v1.17.3/kubernetes.tar.gz
3. 解压kubernetes.tar.gz，进入cluster目录，执行get-kube-binaries.sh
4. 默认从`https://dl.k8s.io`下载二进制包，拼接的下载地址：`https://dl.k8s.io/v1.17.3/kubernetes-server-linux-amd64.tar.gz`，重定向后实际下载地址：`https://storage.googleapis.com/kubernetes-release/release/v1.17.3/kubernetes-server-linux-amd64.tar.gz`

# 安装参考
- 参考1： [文档参考](https://jimmysong.io/kubernetes-handbook/practice/install-kubernetes-on-centos.html)、[源码参考](https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster)
- [参考2](https://github.com/lizhenliang/ansible-install-k8s)
