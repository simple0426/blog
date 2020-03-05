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
* 在pod的弹性伸缩变化过程中，保持外部访问入口不变【service_name、clusterIP】
* 对pod资源访问做负载均衡
* 保持在不同环境下应用部署的拓扑结构和访问方式不变【service_name】

## 实现方式
* service资源基于标签选择器将一组pod定义成一个逻辑组合，并通过自己的ip地址和端口调度代理请求至组内的pod对象之上
* kube-proxy根据apiserver中service和pod的映射关系，在节点上创建相应的iptables、ipvs规则

# 服务发现
- 环境变量方式：在创建pod对象时，kubelet将活动的service对象的一系列环境变量注入到pod中【缺点：仅有那些与创建的pod对象处于同一名称空间且事先存在的service对象才会以环境变量方式注入】
- DNS方式(例如CoreDNS)：集群中创建的每个service对象，都会有ClusterDNS自动生成相关资源记录
  + 拥有ClusterIP的service：A记录【service.namespace.svc.domain==》clusterIP】
  + Headless类型service(无ClusterIP)：A记录【service.namespace.svc.domain==>endpointIP】
  + ExternalName类型service：CNAME记录【service.namespace.svc.domain==》externalName】

# service类型
* ClusterIP：默认的service类型，提供的ip只可以在集群内部访问
* NodePort：建构于clusterIP类型之上，在节点绑定端口；实现方式：【NodePort==》ClusterIP:port】
* LoadBalancer：建构于NodePort类型之上，指向云厂商设置的负载均衡设备(如阿里云SLB)；实现方式：【LoadBalancer==》NodePort==》ClusterIP】
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

## Headless类型service
* 设置：【clusterIp:None】
* 实现：不为service对象创建clusterIP，客户端直接通过pod-ip访问pod
* 实现方式：
  - 有标签选择器(selector)：DNS直接将service_name解析为后端各pod对象的ip；客户端访问时，每次接入的pod资源则是由DNS服务器接收到查询请求时以轮询(roundrobin)方式返回的IP地址
  - 无标签选择器
    + 为ExternalName类型创建CNAME记录，例子如上
    + 对其他类型来说，为那些与当前service共享名称的所有Endpoints对象创建一条记录

# service语法
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

# 负载均衡
kubernetes实现了两种形式的负载均衡：

* TCP负载均衡：工作于传输层的service资源；
  - 基于netfilter之上进行的四层调度，如iptables、ipvs
  - 支持调度http、mysql等应用层服务
* HTTP(S)负载均衡：工作于应用层的Ingress资源
  - 可以实现基于URL的请求调度机制
  - 支持卸载HTTPS中的ssl会话
  - 支持对后端服务器进行健康检查

# Ingress
* 实质：就是一组基于DNS名称(host)或URL路径把请求转发到指定的service资源的规则
* 实现：实现Ingress规则的组件就是Ingress Controller

# Ingress-Controller
* 与k8s核心组件关系：Ingress Controller不是集群kube-controller-manager的一部分，它是k8s集群的一个组件(类似于CoreDNS)，需要单独部署
* 本质：Ingress控制器本身也是作为一个pod资源在集群中运行，它可以由deployment或daemonset创建；由于需要接入外部流量，还需要为其创建相关的NodePort或LoadBalancer类型的Service资源对象
* 实现原理：Ingress控制器基于Ingress定义的规则将流量转发到service对应的后端pod上，service对象仅仅用于辅助Ingress识别pod对象；这种转发机制会绕过service资源，从而省去了kube-proxy实现的端口代理开销
* 实现软件：Ingress控制器可以由任何实现反向代理(HTTP/HTTPS)的软件实现，如nginx、traefik
  - [nginx-ingress][nginx-ingress-doc]
    + [pod][nginx-ingress-pod]
    + [service][nginx-ingress-service]
  - traefik-1.7
    + [官方安装][traefik-offical]
    + [第三方安装][traefik-simple0426]
    + [traefik配置https][traefik-https]

# Ingress语法
- rules：转发规则定义
  + host：匹配的主机名
    * 不支持使用ip地址
    * 不支持端口形式【：port】
    * 字段留空表示匹配所有主机名
- backend：处理请求的后端，rules和backend至少要定义一个
  + serviceName：service名称
  + servicePort：service端口
- tls：包含支持https的对象
  + hosts：使用证书的主机名列表
  + secretName：基于证书创建的secret对象名称

# Ingress范例
* 范例-基于主机名的虚拟主机【traefik.jimmysong.io==》traefik-ingress-service:8080】
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-ingress
  namespace: kube-system
spec:
  rules: 
  - host: traefik.jimmysong.io
    http:
      paths:
      - path: /
        backend:      
          serviceName: traefik-ingress-service   
          servicePort: 8080
```
* 范例-基于URL路径【myapp.abc.com/myapp==》myapp:80】
```
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
      - path: /myapp
        backend:
          serviceName: myapp
          servicePort: 80
```

# Ingress-TLS设置范例
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
  tls:
  - hosts:
    - test.abc.com
    secretName: tomcat-ingress-secret
  rules:
  - host: test.abc.com 
    http:
      paths:
      - path:
        backend:
          serviceName: tomcat-svc
          servicePort: 80
```

[nginx-ingress-doc]: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/
[nginx-ingress-pod]: https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/mandatory.yaml
[nginx-ingress-service]: https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/provider/baremetal/service-nodeport.yaml
[traefik-offical]: https://github.com/containous/traefik/tree/v1.7/examples/k8s
[traefik-simple0426]: https://github.com/simple0426/kubernetes-vagrant-centos-cluster/tree/master/addon/traefik-ingress
[traefik-https]: https://blog.51cto.com/m51cto/2328921
