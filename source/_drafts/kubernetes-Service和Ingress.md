---
title: kubernetes-Service和Ingress
tags:
categories:
---
# service需求背景
* 在pod的弹性伸缩变化过程中，保持外部访问入口不变
* 对pod资源访问做负载均衡
* 保持在不同环境下应用部署的拓扑结构和访问方式不变

# 访问方式
* ClusterIP：默认的service类型，可以在节点或pod内部访问的ip；
    - headless service：配置【clusterIp:None】，则pod通过service_name解析到所有后端的pod—ip供客户端选择
* ExternalName：内部dns,形如service_name.namespace
* NodePort：可以在外部访问的服务端口；k8s会将端口绑定到每个节点上（由于端口浪费，所以实际应用不多）
* LoadBalancer：云厂商设置的负载均衡ip,实现方式：LoadBalancer-->NodePort-->ClusterIP

## 范例
* deployment部署
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
* service部署
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
  - port: 31222             #服务端口：ClusterIP
    protocol: TCP
    targetPort: 80          #容器端口
    nodePort: 32222         #节点端口：NodePort
  selector:                 #选择的pod
    app: nginx
  type: NodePort #端口类型
  # clusterIP: None
```
