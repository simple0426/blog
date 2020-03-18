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

## 一般组成
* Headless Service：为pod资源生成DNS记录
* Statefulset：管控pod资源
* volumeClaimTemplate：为pod资源提供专用且固定的存储

# statefulset范例
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
        # storageClassName: "standard"
        storageClassName: "glusterfs"
        resources:
          requests:
            storage: 2Gi
```

# statefulset管理
## 扩容缩容
>支持扩容缩容，但具体的实现机制依赖于应用本身

* scale方法：`kubectl scale statefulset myapp --replicas=4`
* patch方法：`kubectl patch statefulset myapp -p '{"spec":{"replicas":3}}'`

## 更新
>支持自动更新

* 更新策略：
    - OnDelete：删除pod才会触发重建更新
    - RollingUpdate：自动更新，默认的更新策略
* 分区更新：
  - RollingUpdate也支持分区机制(partition),只有大于索引号(partition)的pod资源才会被滚动更新
  - 若给定的分区号大于副本数量，则意味着不会有pod资源索引号大于此分区号，所有的pod资源均不会被更新
* 命令：
    - 查看镜像信息：`kubectl get pod -o custom-columns=NAME:metadata.name,IMAGE:spec.containers[0].image`
    - 更新镜像：`kubectl set image statefulset myapp myapp=ikubernetes/myapp:v6`
    - 更新状态查询：`kubectl rollout status statefulset myapp`
* 实践
    - 暂存更新操作：将分区号(partition)设置为和副本数(replicas)一样大，此后所有的更新操作都将暂停
    - 金丝雀部署：调整分区号至小于副本数，不断“放出金丝雀”，触发更新操作

## 实践
不同的有状态应用的运维操作过程差别巨大，statefulset本身无法提供通用管理机制  
现实中的各种有状态应用通常是使用专门的自定义控制器专门封装特定的运维操作流程  
这些自定义控制器有时被统一称为operator  

# 范例-etcd集群
## service
```
apiVersion: v1
kind: Service
metadata:
  name: etcd
  labels:
    app: etcd
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  ports:
  - port: 2379
    name: client
  - port: 2380
    name: peer
  clusterIP: None
  selector:
    app: etcd-member
```
## statefulset
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
  labels:
    app: etcd
spec:
  serviceName: etcd
  replicas: 3
  selector:
    matchLabels:
      app: etcd-member
  template:
    metadata:
      name: etcd
      labels:
        app: etcd-member
    spec:
      containers:
      - name: etcd
        image: "quay.mirrors.ustc.edu.cn/coreos/etcd:v3.3.18"
        ports:
        - containerPort: 2379
          name: client
        - containerPort: 2380
          name: peer
        env:
        - name: CLUSTER_SIZE
          value: "3"
        - name: SET_NAME
          value: "etcd"
        volumeMounts:
        - name: data
          mountPath: /var/run/etcd
        command:
          - "/bin/sh"
          - "-ecx"
          - |
            IP=$(hostname -i)
            PEERS=""
            for i in $(seq 0 $((${CLUSTER_SIZE} - 1)));do
                PEERS="${PEERS}${PEERS:+,}${SET_NAME}-${i}=http://${SET_NAME}-${i}.${SET_NAME}:2380"
            done
            exec etcd --name ${HOSTNAME} \
            --listen-peer-urls http://${IP}:2380 \
            --listen-client-urls http://${IP}:2379,http://127.0.0.1:2379 \
            --advertise-client-urls http://${HOSTNAME}.${SET_NAME}:2379 \
            --initial-advertise-peer-urls http://${HOSTNAME}.${SET_NAME}:2380 \
            --initial-cluster-token etcd-cluster-1 \
            --initial-cluster ${PEERS} \
            --initial-cluster-state new \
            --data-dir /var/run/etcd/default.etcd  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: "glusterfs"
      accessModes:
        - "ReadWriteOnce"
      resources:
        requests:
          storage: 2Gi
```
## 验证
* 查看service与pod绑定关系：`kubectl get endpoints -l app=etcd`
* 查看pod：`kubectl get pod -l app=etcd-member -o wide`
* 查看etcd集群状态：`kubectl exec etcd-0 -- etcdctl cluster-health`

## 注意
* 扩缩容：
    - 扩容：先提升副本数量再手动添加节点
    - 缩容：先移除节点再手动缩减副本数量
* 镜像升级：使用scale或patch即可完成应用镜像升级
