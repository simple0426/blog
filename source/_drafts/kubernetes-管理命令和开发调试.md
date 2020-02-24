---
title: kubernetes-管理命令和开发调试
tags:
categories:
---

# kubectl命令
* config: 配置管理
    - 合并多个集群的config文件（clusters、contexts、users）
    - 查看config配置：kubectl config view
    - 查看当前管理的集群：kubectl config get-contexts
    - 切换管理的集群：kubectl config use-context minikube
* edit: 编辑运行中的资源
* get: 查看资源状态
    - kubectl get resource_type resource_name -o wide/yaml/json：获取资源状态，资源类型如下
* describe: 查看资源详情
    - kubectl describe resource_type resource_name
* logs: 查看pod日志
    - kubectl logs pod_id
* create: 创建资源
    - kubectl create -f nginx-deployment.yml
* delete: 删除资源
    - 文件式：kubectl delete -f nginx-deployment.yml
    - 名称式：kubectl delete resource_type resource_name
* scale: pod弹性伸缩
    - kubectl scale rc nginx --replicas=4
* exec: 进入pod或容器执行操作
    - kubectl exec -it pod_name -c container_name /bin/bash

# 调试工具
* [telepresence](https://www.telepresence.io/):基于远程k8s集群的本地开发环境
* [kubectl-debug](https://github.com/aylei/kubectl-debug/blob/master/docs/zh-cn.md):通过启动一个安装了各种排障工具的容器，来诊断目标容器

## kubectl-debug
* [安装](https://github.com/aylei/kubectl-debug/releases)
* 命令使用：kubectl debug pod_name --agentless
* 命令参数：
    - --fork：诊断一个处于CrashLookBackoff的pod
    - --agentless：启动无代理模式【命令使用时自动创建一个agent】
    - --image：指定使用的工具镜像，不使用默认的工具镜像nicolaka/netshoot
    - -c container-name：指定要进入的容器内
