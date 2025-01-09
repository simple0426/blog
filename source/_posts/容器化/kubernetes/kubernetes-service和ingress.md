---
title: kubernetes-service和ingress
date: 2020-03-06 02:12:36
tags:
  - service
  - ingress
  - 服务发现
categories: ['kubernetes']
---

# service
## 实现目标
* 服务发现：在pod的弹性伸缩变化过程中，保持外部访问入口不变
* 负载均衡：定义访问pod资源的策略

## 实现方式
* 服务发现：service定义一个访问入口(service_name、clusterIP)，并基于标签选择器选择一组pod
* 负载均衡：kube-proxy根据service和pod的映射关系(endpoints)，在节点上创建相应的iptables或ipvs规则

## 服务发现的实现

- 环境变量方式：在创建pod对象时，kubelet将活动的service对象的一系列环境变量注入到pod中【缺点：仅有那些与创建的pod对象处于同一名称空间且事先存在的service对象才会以环境变量方式注入】
- DNS方式(例如[CoreDNS](https://github.com/coredns/deployment/tree/master/kubernetes))：集群中的DNS服务会为每个创建的service对象生成相应的解析记录(服务名称：service_name.namespace_name.svc.domain_name)
  + 拥有ClusterIP的service：A记录【服务名称映射clusterIP】
  + Headless类型service(无ClusterIP)：A记录【服务名称映射endpointIP】
  + ExternalName类型service：CNAME记录【服务名称映射externalName】

## 负载均衡的实现

### 实现方式

* iptables：使用灵活、功能强大；但是规则更新和匹配时间随条目数的增加而线性增加
* ipvs：工作在内核态，性能更优异；支持丰富的调度算法

### 参数配置

* 配置：kube-proxy配置--proxy-mode

* kubeadm方式部署集群kube-proxy变更

  ```
  1. 编辑kube-proxy configmap配置：kubectl edit cm kube-proxy -n kube-system 
  2. 重启kube-proxy pod【删除pod后自动重建】
  4. 查看pod日志
  ```

* ipvs配置查看

  * 配置条目：ipvsadm -Ln
  * 使用网卡：kube-ipvs0

# service类型

* ClusterIP：默认的service类型，提供的ip只可以在集群内部访问
* NodePort：建构于clusterIP类型之上，在节点绑定端口，可以在集群外部访问；实现方式：【NodePort==》ClusterIP:port】
* LoadBalancer：建构于NodePort类型之上，指向云厂商设置的负载均衡设备(如阿里云SLB)，可以在集群外部访问；实现方式：【LoadBalancer==》NodePort==》ClusterIP】
* ExternalName：不是定义k8s集群提供的服务，而是把集群外部的某服务以CNAME记录的方式映射到集群内，从而让集群内的pod资源能够访问外部的service;externlName只能是外部服务的域名(不能是ip)
```
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: default
spec:
  type: ExternalName
  externalName: redis.example.com
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379
```

# service语法范例
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
    targetPort: 80          #pod/容器端口
    nodePort: 32222         #节点端口【不设置或设置为0，系统自动配置】
  selector:                 #选择的pod
    app: nginx
  type: NodePort            #端口类型
  # clusterIP: None
```
* 相关deployment
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
  template:         #pod模板
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

# service与ingress对比
* service：工作于传输层(TCP)，构建于pod资源之上
  * 基于netfilter之上进行的四层调度，如iptables、ipvs
  * 支持调度http、mysql等应用层服务

* Ingress：工作于应用层(HTTP/HTTPS)，构建于service资源之上
  - 可以实现基于URL的请求调度机制
  - 支持https证书
  - 支持对后端服务器进行健康检查

# Ingress
* 实质：将访问指定主机名或URL路径的请求转发到特定service资源的规则
* 实现：实现Ingress规则的组件就是Ingress Controller

# Ingress-Controller
* 与k8s核心组件关系：Ingress Controller不是集群kube-controller-manager的一部分，它是k8s集群的一个组件(类似于CoreDNS)，需要单独部署
* 本质：Ingress控制器本身也是作为一个pod资源在集群中运行，它可以由deployment或daemonset创建；由于需要接入外部流量，还需要为其创建相关的NodePort或LoadBalancer类型的Service资源对象
* 实现原理：Ingress控制器基于Ingress定义的规则将流量转发到service对应的后端pod上，service对象仅仅用于辅助Ingress识别pod对象；这种转发机制会绕过service资源，从而省去了kube-proxy实现的端口代理开销
* 实现软件：Ingress控制器可以由任何实现反向代理(HTTP/HTTPS)的软件实现，如nginx、traefik、Istio
  - [ingress-nginx][ingress-nginx-doc]：
    - [资源文件](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/provider/baremetal/deploy.yaml)
    - 资源文件修改
      - 镜像地址修改：registry.cn-hangzhou.aliyuncs.com/simple00426/nginx-ingress-controller:0.34.0
      - 控制器类型修改(确保controller高可用及知晓部署节点)，可选方式例如：Deployment(默认)、DaemonSet
      - 网络及端口设置(将ingress的访问入口暴露给集群外部)，可选方式例如：hostNetwork、hostPort、NodePort(默认)
  - traefik-1.7
    + [官方安装][traefik-offical]
    + [第三方安装][traefik-simple0426]
    + [traefik配置https][traefik-https]
  - Istio：服务治理

# Ingress语法范例
## 基于域名的虚拟主机

```
# traefik.jimmysong.io==》traefik-ingress-service:8080
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-ingress
  namespace: kube-system
spec:
  rules: #转发规则定义
  - host: traefik.jimmysong.io #匹配的主机名【不支持ip、端口形式；字段留空表示所有主机名】
    http:
      paths:
      - path: / 
        backend:      #处理请求的后端
          serviceName: traefik-ingress-service    #后端服务名
          servicePort: 8080  #后端服务端口
```
## 基于URL路径-rewrite

```
# myapp.abc.com/tomcat==》tomcat-service:38080
# myapp.abc.com/nginx==>nginx-service:31222
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-ingress
  annotations:
    traefik.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.abc.com
    http:
      paths:
      - path: /tomcat
        backend:
          serviceName: tomcat-service
          servicePort: 38080
      - path: /nginx
        backend:
          serviceName: nginx-service
          servicePort: 31222
```

## HTTPS设置

* 创建自签名证书
  - openssl genrsa -out tls.key 2048
  - openssl req -new -x509 -key tls.key -out tls.crt -subj /C=CN/ST=Beijing/L=Beijing/O=DevOps/CN=test.abc.com -days 3650
* 将证书放入k8s的secret中：kubectl create secret tls tomcat-ingress-secret --cert=tls.crt --key=tls.key -n testing
* 在ingress中设置
```
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: tomcat
  namespace: testing
spec:
  tls: #包含支持https的对象
  - hosts:
    - test.abc.com #使用证书的主机名列表
    secretName: tomcat-ingress-secret #基于证书创建的secret对象名称
  rules:
  - host: test.abc.com 
    http:
      paths:
      - path:
        backend:
          serviceName: tomcat-svc
          servicePort: 80
```

## 个性化设置-[annotation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/)

```
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: tomcat-https
  annotations:
    kubernetes.io/ingress.class: "nginx" #部署多种ingress控制器时选择使用
    nginx.ingress.kubernetes.io/ssl-redirect: 'true'
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
```

# ingress-controller高可用

* 选用daemonset或deployment在多个节点部署多个pod
* 通过标签/标签选择器、污点/容忍度选定特定的几个节点方便前端代理
* 前端代理使用nginx+keepalived

[ingress-nginx-doc]: https://kubernetes.github.io/ingress-nginx/
[traefik-offical]: https://github.com/containous/traefik/tree/v1.7/examples/k8s
[traefik-simple0426]: https://gitee.com/simple0426/kubernetes-vagrant-centos-cluster/tree/master/addon/traefik-ingress
[traefik-https]: https://blog.51cto.com/m51cto/2328921
