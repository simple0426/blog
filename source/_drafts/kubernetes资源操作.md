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
