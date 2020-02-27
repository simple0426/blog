---
title: kubernetes-监控和日志
tags:
categories:
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

|         -         |           API           |                        注释                        |
|-------------------|-------------------------|----------------------------------------------------|
| Resources metrics | metrics.k8s.io          | 主要的实现为metrics-server，提供资源监控           |
| Custom metrics    | custom.metrics.k8s.io   | 主要的实现为prometheus，提供资源监控和自定义监控   |
| External metrics  | external.metrics.k8s.io | 主要的实现为云厂商的provider，提供云资源的监控指标 |


## 资源指标API
* metrics-server部署： https://github.com/kubernetes-sigs/metrics-server
* 验证资源指标API可用性
    - kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods"
    - kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
* 获取node或pod对象的资源使用情况；kubectl top node/pod

# prometheus监控
## 架构
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/prometheus-architecture.png)

* 数据采集(pull模型)：pushgateway(short-lived jobs)、exporter、service discovery
* 数据聚集存储：node-hdd/ssd(storage)、promQL(查询语言)
* 报警：altermanager
* 展示：api clients、grafana、web UI
* 与k8s集成：基于prometheus收集和存储指标数据，借助于k8s-prometheus-adapter将这些指标数据查询接口转换为标准的kubernetes自定义指标

## 插件
* kube-eventer：阿里开源的k8s事件离线通知工具
* 各类exporter
    - node_exporter：收集主机指标数据
    - kubelet/ccadvisor：收集容器数据
    - APIServer：收集APIServer的性能指标数据
    - etcd：收集etcd存储集群的指标数据
    - kube-state-metrics：能通过API Service监听资源对象的状态并生成相关的指标；包含资源类型相关的计数器和元信息，包含指定类型对象总数、资源限额、容器状态、pod资源标签
* 各类api clients

## 部署
* [kubernetes上部署prometheus](https://github.com/coreos/prometheus-operator)

# 日志分类
* 主机内核日志：网络栈异常、驱动异常、文件系统异常
* runtime日志：docker日志，可以排查删除pod hang等问题
* 核心组件日志：
    - apiserver日志用来审计
    - scheduler日志可以诊断调度
    - etcd日志可以查看存储状态
    - ingress日志可以分析接入层流量
* 部署应用的日志：查看业务层状态

# 日志采集
* 方案1：
    - fluentd:采集、聚合
    - elsticsearch：索引、存储
    - kibanna：可视化、分析
* 方案2：
    - fluentd:采集、聚合
    - influxdb：索引、存储
    - grafana：可视化、分析

# 阿里云监控实践
* 云监控
* SLS：日志服务
* ARMS(java/php)：性能监控
* AHAS(应用高可用服务)：
    - 架构感知
    - 故障演练
    - 流控降级
