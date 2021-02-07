---
title: kubernetes-监控和日志
tags:
  - 日志
  - 监控
  - prometheus
  - metrics-server
categories:
  - kubernetes
date: 2020-03-11 17:03:40
---

# k8s监控指标

* 节点对象：系统指标、CPU内存及网络流量
* k8s资源对象：k8s事件
* 容器pod对象：容器本身的系统指标，同样有CPU内存等指标
* 应用性能统计：QPS统计、jvm数据收集
* 应用业务统计

# 监控方案

## 早期方案-heapster

* 架构
  * 数据采集：kubelet/cadvisor
  * 数据接口：heapster、kubelet、summary、prometheus
  * 数据存储：inflexDB
  * 数据消费：dashboard、horizontal pod autoscaler(HPA/pod弹性伸缩)、kubectl top、grafana(可视化)

* heapter缺点
  * 包含了太多的存储后端接口代码，某些代码已不再维护
  * 接口格式混乱，不支持prometheus
  * 仅支持以CPU为基础的弹性伸缩

## 新一代监控方案

* 资源指标API和自定义指标API被创建为API定义而非具体实现，他们作为聚合的API安装到kubernetes集群中，从而允许API在保持不变的情况下切换其具体的实现方案
* 通过APIServer Aggregated API注册了三种不同的metrics接口，将监控的消费能力进行标准化和解耦，从而实现了与社区的融合 

| -                 | API                     | 注释                                               |
| ----------------- | ----------------------- | -------------------------------------------------- |
| Resources metrics | metrics.k8s.io          | 主要的实现为metrics-server，提供资源指标采集       |
| Custom metrics    | custom.metrics.k8s.io   | 主要的实现为prometheus，提供自定义指标采集         |
| External metrics  | external.metrics.k8s.io | 主要的实现为云厂商的provider，提供云资源的监控指标 |

# API聚合层

## 介绍

在kubernetes 1.7版本引入聚合层，允许第三方应用程序将自己注册到apiserver上，仍然通过apiserver的 HTTP URL对新的API进行访问和操作。为了实现这个机制，kubernetes在kube-apiserver服务中引入了一个API聚合层，用于将扩展的API的访问请求转发到用户服务的功能。当你访问apis/metrics.k8s.io/v1beta1的时候，实际上访问的是一个叫做kube-aggregator的代理。kube-apiserver是代理的一个后端，而Metrics server是另一个后端。通过这种方式，我们就可以很方便的扩展kubernetes的API了

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/kube-aggergation.png)

## 功能启用

在kube-apiserver中添加启动参数用以开启API聚合层功能

```
--enable-aggregator-routing=true \
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \
--requestheader-allowed-names=kubernetes \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--proxy-client-cert-file=/opt/kubernetes/ssl/server.pem \
--proxy-client-key-file=/opt/kubernetes/ssl/server-key.pem \
```

--requestheader-allowed-names：由ca签署的客户端证书的CN名称（可以是apiserver、kube-proxy、kubectl等证书的CN名称）

# metrics-server

## 原理

metrics server从每个节点上的kubelet(cadvisor)公开的摘要API收集指标数据，然后通过kubernetes聚合器注册在master的APIServer中

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/hpa-metrics-sever.png)

## 部署

项目地址：https://github.com/kubernetes-sigs/metrics-server 

使用前提：API Server开启聚合层功能

下载资源文件：

```
curl -Lo metrics-server.yaml https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.4.1/components.yaml
```

修改资源文件

```
  containers:
  - name: metrics-server
    image: registry.cn-hangzhou.aliyuncs.com/simple00426/metrics-server-amd64:v0.4.1
    imagePullPolicy: IfNotPresent
    args:  
      - --cert-dir=/tmp
      - --secure-port=4443
      - --kubelet-insecure-tls
      - --kubelet-preferred-address-types=InternalIP
```

- 修改镜像地址(image)：registry.cn-hangzhou.aliyuncs.com/simple00426/metrics-server-amd64:v0.4.1
- 设置容器启动参数(args)
  + `--kubelet-insecure-tls`：metrics server连接kubelet(作为服务端)时，不对kubelet证书的ca进行校验
  + `--kubelet-preferred-address-types=InternalIP`：metrics server通过内部ip（kubectl describe node|grep InternalIP）访问kubelet，获取指标数据

可通过Metrics API在Kubernetes中获得资源使用率指标，例如容器CPU和内存使用率。这些度量标准既可以由用户直接访问（例如，通过使用`kubectl top`命令），也可以由集群中的控制器（例如，Horizontal Pod Autoscaler）用于进行决策。

```
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl top node/pod
kubectl get apiservice
```

# prometheus

## prometheus技术栈组件

* prometheus server：收集、存储、查询监控指标
* 客户端组件
  * cadvisor(kubelet)：采集容器cpu、内存使用情况；PromQL查询关键词：`container_`
  * node_exporter：采集linux节点cpu、内存使用情况；PromQL查询关键词：`node_`
  * kube-state-metrics：通过API Server收集资源对象的状态并生成相关的指标；PromQL查询关键词：`kube_`
* altertmanager：_告警规则需要在prometheus中配置_
* pushgateway：用于支持短周期任务push数据的网关
* grafana：度量指标可视化

## k8s中部署prometheus方式

* 分组件部署：https://github.com/simple0426/sysadm/tree/master/kubernetes/prometheus

* [kube-prometheus](https://github.com/coreos/kube-prometheus)：提供了基于prometheus operator和prometheus的完整集群监控技术栈的示例配置，包含如下
  * [Prometheus Operator](https://github.com/coreos/prometheus-operator)：设置在k8s集群中运行的prometheus，以获取k8s资源对象
  * 高可用[Prometheus](https://prometheus.io/)
  * 高可用[Alertmanager](https://github.com/prometheus/alertmanager)
  * [Prometheus node-exporter](https://github.com/prometheus/node_exporter)
  * [Prometheus Adapter for Kubernetes Metrics APIs](https://github.com/DirectXMan12/k8s-prometheus-adapter)：prometheus实现k8s资源监控(metrics-server)的适配器
  * [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)：收集k8s资源对象的相关信息
  * [Grafana](https://grafana.com/)
* [stable/prometheus-operator](https://github.com/helm/charts/tree/master/stable/prometheus-operator)：helm社区维护的prometheus监控技术栈，类似于kube-prometheus

## kube-prometheus部署

* 设置api-server开启聚合层功能：--enable-aggregator-routing=true
* 设置kubelet【/etc/sysconfig/kubelet】
  - `--authentication-token-webhook=true`
  - `--authorization-mode=Webhook`
* [下载资源文件](https://github.com/coreos/kube-prometheus/tree/master/manifests)
  - `kubectl create -f manifests/setup`
  - `kubectl create -f manifests/`

# 日志分类

> 宿主机/k8s组件、pod标准输出日志可以通过在k8s中部署[filebeat](https://github.com/elastic/beats/blob/master/deploy/kubernetes/filebeat-kubernetes.yaml)收集

* 宿主机和核心组件日志，通过hostpath挂载/var/log/messages收集日志
    - apiserver日志用来审计
    - scheduler日志可以诊断调度
    - etcd日志可以查看存储状态
    - ingress日志可以分析接入层流量
* 应用程序日志
    * 标准输出(docker engine写日志)：/var/lib/docker/containers/<Container-id>/<container-id>-json.log
    * 日志文件

# 程序日志收集

* 将pod中的日志挂载到宿主机的目录(hostPath)，然后使用filebeat收集；解决多个pod部署在同一宿主机日志混乱的方案如下
  * 【开发】在程序中根据容器名称命名日志文件名，例如/tmp/nginx-1.log、/tmp/nginx-2.log
  * 【运维】在pod中的hostPath中根据容器名称，挂载到宿主机的不同目录；例如：/tmp/nginx-1/access.log、/tmp/nginx-2/access.log

* 每一个pod中都部署一个日志收集容器，使用emptyDir共享日志目录让日志收集程序读取到
* 【非k8s解决方式】应用程序直接推送日志，比如推送到专用日志服务器

| 方式                         | 优点                                                     | 缺点                                                         |
| ---------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| 在node上部署一个日志收集程序 | 每个node仅部署一个日志收集程序；资源消耗少；对应用无侵入 | 应用程序日志如果写到标准输出和标准错误输出，那就不支持多行日志（docker使用json-log日志驱动） |
| pod中附加专用日志收集容器    | 低耦合                                                   | 每个pod中都增加一个日志收集代理，增加资源消耗和运维维护成本  |
| 应用程序直接推送日志         | 无需额外工具                                             | 浸入应用，增加应用复杂度                                     |

# 日志处理方案

> [k8s中部署efk范例](https://github.com/simple0426/sysadm/tree/master/kubernetes/elk)

* 数据采集：filebeat、fluentd
* 过滤清洗：logstash
* 消息队列：redis、kafka
* 索引存储：elasticsearch、influxDB
* 可视化分析：kibana、grafana

