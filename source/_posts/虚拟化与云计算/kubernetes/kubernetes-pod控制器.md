---
title: kubernetes-pod控制器
tags:
  - Deployment
  - Job
  - CronJob
  - DaemonSet
categories:
  - kubernetes
date: 2020-02-24 19:32:25
---

# Deployment
## 使用场景
* 保证集群内可用pod的数量
* 为所有pod更新镜像版本
* 更新的过程中保证服务可用性
* 更新过程中出现问题快速回滚

## 管理模式与架构
* Deployment只管理不同版本的replicaset
* 由replicaset管理pod副本数量
* 关系链

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/k8s-deployment.jpg)
## 关键配置项
* spec字段
    - revisionHistoryLimit: 保留历史revision数量
    - progressDeadlineSeconds: 判断Deployment status condition为failed的最大时间
* 升级策略
    - maxSurge: 升级过程中最多存在多少个超过期望数量的pod
    - maxUnavailable: 升级过程中最多有多少个pod不可用

## 范例
* 文件
```
apiVersion: apps/v1 
kind: Deployment
metadata:       #Deployment元信息
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3       #期望的pod数量
  selector:         #pod选择器
    matchLabels:
      app: nginx
  template:      #pod模板
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.12.2
        ports:
        - containerPort: 80
```
* 更新与回滚
    - 更新容器镜像：kubectl set image deployment nginx-deployment nginx=nginx:1.13
    - 【注释】kubectl set image 资源类型(deployment) 资源名称(nginx-deployment) 容器名称(nginx) 镜像名称(nginx:1.13)
    - 回滚镜像到上一版本：kubectl rollout undo deployment nginx-deployment
* pod弹性伸缩：kubectl scale --replicas=2 deployment/nginx-deployment
    - scale命令支持对Deployment, ReplicaSet, Replication Controller, or StatefulSet进行扩容、缩容
* 服务发布
    - 通过expose方法，建立service将容器端口绑定到宿主机端口：kubectl expose deployment nginx-deployment --type=NodePort
    - service查看绑定关系：kubectl get svc
    - url访问地址：http://192.168.99.100:30345/

# Job
相当于linux系统下的定时任务at/crontab
## 使用场景
* 创建一定数量的pod，并保证可以成功地运行终止
* 跟踪pod状态，并根据重试策略进行失败重试
* 确定依赖关系，保证任务依次执行
* 控制任务并发度

## 范例
* 一次性任务
```
apiVersion: batch/v1
kind: Job #job类型
metadata: #元信息
  name: pi
spec:
  template: #pod模板
    spec:
      containers:
      - name: pi
        image: perl:slim
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never # 重试策略，其他类型为：Onfailure、Always
  backoffLimit: 4 #重试次数
```
* 并发参数
```
apiVersion: batch/v1
kind: Job
metadata:
  name: paral
spec:
  completions: 8 #执行的pod总数
  parallelism: 2 #并行执行的pod数
  template:
    spec:
      containers:
      - name: param 
        image: ubuntu
        command: ["/bin/sh"]
        args: ["-c", "sleep 30; date"]
      restartPolicy: OnFailure #失败重启
```
* 周期任务CronJob(相当于linux中的crontab)
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *" #周期参数，与crontab相同
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the kubernetes cluster
          restartPolicy: OnFailure
  startingDeadlineSeconds: 10 #最长启动时间
  concurrencyPolicy: Allow #是否允许并行运行(解决情况：任务的执行时间超过周期间隔)
  successfulJobsHistoryLimit: 3 #允许留存的成功job数
```

# DaemonSet
相当于linux的守护进程，例如supervisor
## 使用场景
* 保证集群中的每个节点都运行一个相同的pod
* 节点加入或删除时，节点部署或删除对应的pod
* 应用场景
    - 集群存储进程：glusterd，ceph
    - 日志收集进程：fluentd，logstash
    - 节点状态监控进程

## 范例
* 文件
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: fluent/fluentd:v1.4-1
        resources: #资源设置
          limits:
            memory: 200Mi
          requests:
            cpu: 100m #相当于0.1cpu
            memory: 200Mi
```
* 更新镜像：kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=fluent/fluentd:v1.4
* 回滚：kubectl rollout undo ds/fluentd-elasticsearch
    - 中间状态：kubectl rollout status ds/fluentd-elasticsearch
