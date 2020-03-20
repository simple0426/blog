---
title: kubernetes-pod管理
tags:
categories:
---

# 资源对象
>命令查看：kubectl get pod pod-xxx -o yaml

* spec：期望的状态
* status：观测到的状态
* metadata
    - labels：识别资源的标签
    - annotation：描述资源的注解
    - OwnerReference：描述多个资源之间的相互关系

## labels
* 定义：标识型的key:value元数据
    - environment: production
    - release: stable
* 作用：用于筛选资源，唯一的组合资源的方法
* 被调用：可以使用selector来查询
    - 相等型selector: Tie=front,Env=dev
    - 集合型selector：Env in (test,gray)，Tie notin (front,back)，release，!release

## annotations
* 定义：标识型的key:value信息
* 作用：
    - 存储资源的非标识信息
    - 扩展资源的spec/status
* 特点
    - 一般比label更大
    - 可以包含特殊字符
    - 可以结构化(如json)也可以非结构化

## OwnerReference
* 一个ReplicationController、replicaset、statefulset、daemonset、deployment是一组pod的owner
* 通过OwnerReference，可以反向查询创建资源的对象，方便进行级联删除

## label管理
* 显示label：kubectl get pod --show-lables
* 添加label：kubectl label deployment nginx-deployment time=0217
    - 强制覆盖：kubectl label deployment nginx-deployment time=0217 --overwrite
* 删除label：kubectl label deployment nginx-deployment time-
* 通过selector选择label：
    - 相等型：kubectl get deployment --show-labels -l env=test
    - 集合型：kubectl get deployment --show-labels -l "env in (test)"
