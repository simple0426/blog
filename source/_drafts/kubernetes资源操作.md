---
title: kubernetes资源操作
tags:
categories:
---

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
