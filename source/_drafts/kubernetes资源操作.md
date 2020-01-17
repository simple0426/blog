---
title: kubernetes资源操作
tags:
categories:
---
以下为通过api可以操作的k8s资源对象，也是kube-dashboard页面上workloads的内容
# pod
pod是k8s任务调度的基本单元

* 文件pod_nginx.yml
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```
* 命令创建：kubect create -f pod_nginx.yml
* 查看存在的pod：kubectl get pod
* 查看pod详情：kubectl describe pod nginx
* 删除pod：kubectl delete -f pod_nginx.yml

# ReplicationController
pod的复制抽象，用于解决pod的扩容/缩容问题

* 文件rc_nginx.yml
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
* 查看rc：kubectl get rc
* 查看rc详情：kubectl describe rc
* 调整容器数量：kubectl scale rc nginx --replicas=4
* 删除rc：kubectl delete -f rc_nginx.yml

# ReplicaSet
下一代的ReplicationController，支持基于集合的选择器(selector)

* 文件rs_nginx.yml
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      name: nginx
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
* 查看rs：kubectl get rs
* 查看rs详情：kubectl describe rs
* 调整容器数量：kubectl scale rs nginx --replicas=4
* 删除rs：kubectl delete -f rs_nginx.yml

# Deployment
Deployment支持对pod和ReplicaSet的动态更新，创建deployment后会包含replicaset及相应的pod

* 文件
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
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
* 更新系统镜像：kubectl set image deployment nginx-deployment nginx=nginx:1.13
* 回滚镜像到上一版本：kubectl rollout undo deployment nginx-deployment
* 通过expose方法，建立service将容器端口绑定到宿主机端口：kubectl expose deployment nginx-deployment --type=NodePort
* service查看绑定关系：kubectl get svc
* url访问地址：http://192.168.99.100:30345/

# service
## 类型
* ClusterIP：可以在节点或容器内部访问的ip；容器或pod的ip会发生变化，但是集群ip不变
    - 可以通过expose命令将pod发布为service，从而使pod具有clusterip，例如：kubectl expose deployment nginx-deployment
* NodePort：可以在外部访问的服务端口；k8s会将端口绑定到每个节点上（由于端口浪费，所以实际应用不多）
    - 可以通过expose命令（--type='NodePort'），将pod发布为service，例如：kubectl expose pod nginx-pod --type='NodePort'
* LoadBalancer：云厂商设置的负载均衡ip
* ExternalName：内部dns

## 文件方式创建
* 创建pod
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - name: nginx-port
      containerPort: 80
```
* 基于pod创建service
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
  - port: 31222                 #服务端口：ClusterIP
    nodePort: 32222         #节点端口：NodePort
    targetPort: nginx-port #容器端口
    protocol: TCP
  selector:
    app: nginx
  type: NodePort
```
