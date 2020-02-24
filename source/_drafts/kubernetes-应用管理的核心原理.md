---
title: kubernetes-应用管理的核心原理
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

# 控制器
>控制循环

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/k8s-control-circle.jpg)

# API设计
* 声明式
    - 天然记录了状态
    - 幂等操作，可在任意时刻反复操作
    - 正常操作即巡检
    - 可合并多个变更
* 命令式
    - 如果命令没有响应，需要反复重试、记录当前的操作
    - 如果多次重试、操作有可能出问题
    - 需要单独的巡检操作，巡检本身也可能会有问题或带来额外影响
    - 多并发操作，需要加锁处理；不仅复杂，也降低了系统执行效率

# 控制器模式
>API->Controller

* 由声明式的API驱动(k8s资源对象)
* 由控制器异步地控制系统向终态趋近
* 使系统自动化和无人值守成为可能
* 便于扩展(自定义资源和控制器)

# 身份认证和安全
* 集群中pod的自我身份认证：ServiceAccount
* 容器运行安全管控(类似linux中的selinux)：sepc.containers：securitycontext

# securityContext
securityContext主要用于容器运行时的安全管控，类似linux的selinux、sudo等内容，通过限制容器的行为，从而保障系统和其他容器的安全
## 选项位置
* 容器级别的securityContext：对指定容器生效
* pod级别的securityContext：对指定pod中的所有容器有效
* Pod Security Policies(PSP):对集群内的所有pod有效

## 选项类别
* 根据用户id和组id控制访问对象
* 基于selinux的安全标签
* 以特权或非特权方式允许(privileged)
* 是否能够权限升级(allowPrivilegeEscalation)
* 通过Linux Capablities为其提供部分特权
* 基于seccomp过滤进程可以操作的系统调用

## 范例
>以uid为1000的非特权用户允许容器，并禁止提权

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-securitycontext
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["/bin/sh", "-c", "sleep 86400"]
      securityContext: #容器级别securitycontext
        runAsNonRoot: true
        runAsUser: 1000
        allowPrivilegeEscalation: false
```
