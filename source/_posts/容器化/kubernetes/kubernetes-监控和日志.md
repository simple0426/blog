---
title: kubernetes-监控和日志
tags:
  - metrics-server
  - prometheus
  - 监控
  - 日志
categories:
  - kubernetes
date: 2020-03-11 17:03:40
---

# 监控类型
* 资源监控(cpu、内存、磁盘、带宽)
* 性能监控-APM(jvm)
* 安全监控
* k8s事件监控

# 监控方案-heapster
## 架构
* 数据采集：kubelet/cadvisor
* 数据接口：kubelet、summary、prometheus
* 数据消费：dashboard、horizontal pod autoscaler(HPA/pod弹性伸缩)、kubectl top

## 整合方案
* inflexDB：存储
* grafana：可视化
* heapster：数据采集

## heapter缺点
* 包含了太多的存储后端接口代码，某些代码已不再维护
* 接口格式混乱，不支持prometheus
* 仅支持以CPU为基础的弹性伸缩

# 新一代监控方案
* 资源指标API和自定义指标API被创建为API定义而非具体实现，他们作为聚合的API安装到kubernetes集群中，从而允许API在保持不变的情况下切换其具体的实现方案
* 通过APIServer Aggregated API注册了三种不同的metrics接口，将监控的消费能力进行标准化和解耦，从而实现了与社区的融合 

| -                 | API                     | 注释                                                         |
| ----------------- | ----------------------- | ------------------------------------------------------------ |
| Resources metrics | metrics.k8s.io          | 主要的实现为[metrics-server](https://github.com/kubernetes-sigs/metrics-server)，提供资源监控 |
| Custom metrics    | custom.metrics.k8s.io   | 主要的实现为prometheus，提供资源监控和自定义监控             |
| External metrics  | external.metrics.k8s.io | 主要的实现为云厂商的provider，提供云资源的监控指标           |

# prometheus

## 监控指标

* node节点状态、资源利用率
* 资源对象(serivce/deployment/pod)状态、数量
* 容器资源利用率
* 应用程序
  * 程序性能数据，例如java项目jvm exporter
  * 业务数据

## 架构组件

* prometheus
* 客户端组件
  * cadvisor(kubelet)：采集容器cpu、内存使用情况；PromQL查询关键词：`container_`
  * node_exporter：采集linux节点cpu、内存使用情况；PromQL查询关键词：`node_`
  * kube-state-metrics：通过API Server收集资源对象的状态并生成相关的指标；PromQL查询关键词：`kube_`
* altertmanager【在prometheus中配置告警规则】
* pushgateway：用于支持短周期任务push数据的网关
* grafana：[dashboard模板](https://grafana.com/grafana/dashboards)

## 快速部署

* [kube-prometheus](https://github.com/coreos/kube-prometheus)：提供了基于prometheus operator和prometheus的完整集群监控技术栈的示例配置，包含如下
  * [Prometheus Operator](https://github.com/coreos/prometheus-operator)：设置在k8s集群中运行的prometheus，以获取k8s资源对象
  * 高可用[Prometheus](https://prometheus.io/)
  * 高可用[Alertmanager](https://github.com/prometheus/alertmanager)
  * [Prometheus node-exporter](https://github.com/prometheus/node_exporter)
  * [Prometheus Adapter for Kubernetes Metrics APIs](https://github.com/DirectXMan12/k8s-prometheus-adapter)：prometheus实现k8s资源监控(metrics-server)的适配器
  * [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)：收集k8s资源对象的相关信息
  * [Grafana](https://grafana.com/)
* [stable/prometheus-operator](https://github.com/helm/charts/tree/master/stable/prometheus-operator)：helm社区维护的prometheus监控技术栈，类似于kube-prometheus

# 日志分类

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

# 日志采集方案

* 数据采集：filebeat、fluentd
* 过滤清洗：logstash
* 消息队列：redis、kafka
* 索引存储：elasticsearch、influxDB
* 可视化分析：kibana、grafana

