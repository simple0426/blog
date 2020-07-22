---
title: kubernetes-statefulset控制器
tags:
  - StatefulSet
categories:
  - kubernetes
date: 2020-03-18 23:57:38
---

# 有无状态
* 分类标准：应用程序在和其他用户、设备、程序通信时，根据是否需要记录相关状态信息以用于下次通信，可以将程序分类如下：
    - 有状态应用(stateful)：需要记录信息
    - 无状态应用(stateless)：无需记录
* 控制器
    - 无状态控制器：replicaset，多个pod使用共享的存储卷(ReadWriteMany，ReadMany)
    - 有状态控制器：statefulset，每个pod使用专用的存储卷(ReadWriteOnce)
* 状态和存储交叉组合
    - 需要读写磁盘的有状态应用：支持事务功能的RDBMS；各种分布式存储系统（redis cluster、mongodb、zookeeper）
    - 需要读写磁盘的无状态应用：具有幂等性的文件上传服务；从外部存储加载静态资源以响应用户请求的web服务
    - 无磁盘访问的无状态应用：地理坐标转换器
    - 无磁盘访问的有状态应用：电子商城中的购物车系统

# statefulset介绍
## 特点
* 稳定且唯一的网络标识符：pod名称
* 稳定且持久的存储：基于动态或静态的pvc
* 有序的部署和终止：基于索引号从前往后部署，基于索引号从后往前终止
* 有序的自动滚动更新：基于索引号从后往前更新

## 组成要素
* service对象：Headless Service（为pod资源生成DNS记录）
  * 设置：【clusterIp:None】
  * 实现：不为service对象创建clusterIP，DNS服务直接将service_name解析为后端各pod对象的名称和ip，客户端直接通过pod名称或ip访问pod
  * 解析记录：【_service-name_===>_statefulset-name-{index}_._service-name_._namespace-name_.svc._domain-name_】
* StatefulSet对象：管控pod资源
  * serviceName：关联headless service
  * volumeClaimTemplates：为pod资源提供专用且固定的存储

# statefulset创建
```
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  labels:
    app: myapp-svc
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: myapp-pod
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp
spec:
  updateStrategy:
    rollingUpdate:
      partition: 2
  serviceName: myapp-svc
  replicas: 3
  selector:
    matchLabels:
      app: myapp-pod
  template:
    metadata:
      labels:
        app: myapp-pod
    spec:
      containers:
        - name: myapp
          image: ikubernetes/myapp:v5
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: myappdata
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: myappdata
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "glusterfs"
        resources:
          requests:
            storage: 2Gi
```

# statefulset管理
## 扩容缩容
>支持扩容缩容，但具体的实现机制依赖于应用本身

`kubectl scale statefulset myapp --replicas=4`

## 镜像更新
>支持滚动更新

* 更新策略：
    - OnDelete：删除pod才会触发重建更新
    - RollingUpdate：滚动更新，默认的更新策略
      - 默认的滚动更新方式
      - 分区更新(partition)：只有大于索引号(partition)的pod资源才会被滚动更新；若给定的分区号大于副本数量，则所有的pod资源均不会被更新
* 命令：
    - 查看镜像信息：`kubectl get pod -o custom-columns=NAME:metadata.name,IMAGE:spec.containers[0].image`
    - 更新镜像：`kubectl set image statefulset myapp myapp=ikubernetes/myapp:v6`
    - 更新状态查询：`kubectl rollout status statefulset myapp`
* 实践
    - 暂存更新操作：将分区号(partition)设置为和副本数(replicas)一样大，此后所有的更新操作都将暂停
    - 金丝雀部署：调整分区号至小于副本数，不断“放出金丝雀”，触发更新操作

# 最佳实践

不同的有状态应用的运维操作过程差别巨大，statefulset本身无法提供通用管理机制  
现实中的各种有状态应用通常是使用专门的自定义控制器专门封装特定的运维操作流程  
这些自定义控制器有时被统一称为operator  

## 范例-etcd

* 官方operator：https://github.com/coreos/etcd-operator
* 第三方制作：https://github.com/simple0426/k8s-statefulset