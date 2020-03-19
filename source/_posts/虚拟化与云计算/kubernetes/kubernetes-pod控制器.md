---
title: kubernetes-pod控制器
tags:
  - Deployment
  - Job
  - CronJob
  - DaemonSet
  - Replicaset
categories:
  - kubernetes
date: 2020-02-24 19:32:25
---

# pod控制器
## 需求来源
* 自主式pod被调度到节点后，由kubelet负责容器的存活性；
  - 容器主进程崩溃后，kubelet自动重启相应的容器；
  - 非主进程崩溃类型的容器错误，需要用户自定义存活性探测(liveness probe),以便kubelet可以探测到此类故障
* pod意外删除或节点故障，需要节点以外的pod控制器负责其重建

## 原理
* pod控制器由master的kube-controller-manager组件提供，确保各资源的当前状态(status)匹配用户期望的状态(spec)
* 常见控制器(workload)：ReplicationController、Replicaset、Deployment、Daemonset、Job、CronJob、Statefulset

## 语法定义
* 标签选择器(selector)
* 期望的副本数量(replicas)：scale命令支持对Deployment, ReplicaSet, Replication Controller, or StatefulSet进行扩容、缩容
* pod模板(template)
* minReadySeconds：新建pod等待多久才会将其视为就绪可用

# Replicaset
它是新版本的ReplicationController，相较于RC，它支持基于集合的标签选择器(set-based)
## 与自主式pod区别
* 确保pod资源对象数量符合期望值
* 确保pod健康运行(节点故障时被调度到其他节点运行)
* 弹性伸缩

## 范例
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-example
spec:
  replicas: 2 #副本数
  selector: #pod模板选择器，包含mathLabels和matchExpressions两种
    matchLabels:
      app: rs-demo
  template: # pod模板
    metadata:
      labels: #模板标签
        app: rs-demo
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v2
        ports:
        - name: http
          containerPort: 80
```

## 变更操作
>均可以通过修改资源文件，使用命令kubectl apply/replace应用修改

* 副本数量
  - 影响：可以随时修改，并能实时生效
  - 命令：`kubectl scale  rs rs-example --replicas=5 --current-replicas=3`
* 标签选择器【一般不操作】
  - 影响：修改后，可能会影响控制器和pod对象的映射关系，使pod对象不受控制器管理
  - 命令：`kubectl label pod rs-example-5s745 app= --overwrite`
* pod模板
  - 影响：修改后，仅对新建的pod副本有影响
  - 修改镜像文件命令：`kubectl set image rs rs-example  myapp=ikubernetes/myapp:v2`

# Deployment
## 定义
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/k8s-deployment.jpg)

* Deployment构建于replicaset之上，大部分功能由replicaset控制器实现，Deployment管理不同版本的replicaset
* 相较于Replicaset，新增的功能有：
  - 可以查看Deployment对象的升级过程
  - 可以保存对Deployment的每一次修改，出现问题时可以执行回滚操作
  - 可以随时暂停和启动更新
  - 多种自动更新方案
    + Recreate：全面停止、删除旧的pod后用新版本替代
    + RollingUpdate：滚动更新，逐步替换旧的pod至新版本

## 关键配置项
```
spec字段
  - revisionHistoryLimit: 保留历史revision数量
  - progressDeadlineSeconds: 判断Deployment status condition为failed的最大时间
strategy:
  rollingUpdate:
    maxSurge: 25% 升级过程中最多存在多少个超过期望数量的pod
    maxUnavailable: 25% 升级过程中最多有多少个pod不可用
  type: RollingUpdate【升级策略】
```

## 范例
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

## 变更操作
* 滚动更新镜像：`kubectl set image deployment nginx-deployment nginx=nginx:1.13`
* 金丝雀部署：
  1. 设置maxSurge=1，maxUnavailable确保升级过程中仅生成一个新的pod
  2. 在升级镜像之后，暂停更新：`kubectl set image deployment nginx-deployment nginx=nginx:1.13 && kubectl rollout pause deploy nginx-deployment`
  3. 通过service或ingress及相关策略路由将一部分流量导入新的pod进行验证
  4. 验证通过后恢复滚动升级：`kubectl rollout resume deploy nginx-deployment`
* 查看滚动更新状态:`kubectl rollout status deploy nginx-deployment`
* 回滚镜像到上一版本：`kubectl rollout undo deployment nginx-deployment`
* 扩容缩容：kubectl scale --replicas=2 deployment/nginx-deployment

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
