---
title: kubernetes-安全
tags:
categories:
---
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
