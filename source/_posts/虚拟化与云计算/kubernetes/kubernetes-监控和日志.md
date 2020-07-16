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

|         -         |           API           |                        注释                        |
|-------------------|-------------------------|----------------------------------------------------|
| Resources metrics | metrics.k8s.io          | 主要的实现为metrics-server，提供资源监控           |
| Custom metrics    | custom.metrics.k8s.io   | 主要的实现为prometheus，提供资源监控和自定义监控   |
| External metrics  | external.metrics.k8s.io | 主要的实现为云厂商的provider，提供云资源的监控指标 |

# metric-server部署
安装参考：https://github.com/kubernetes-sigs/metrics-server  
错误处理：以下解决方式适用于手动安装的kubernetes集群（非kubeadm方式）  

## 安装metric-server证书
>错误：metric-server x509: certificate signed by unknown authority

* [需要先安装cfssl工具](https://pkg.cfssl.org/)
* cd /etc/kubernetes/pki/
* 创建签名请求
```
cat > metrics-server-csr.json <<EOF
{
  "CN": "aggregator",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Hangzhou",
      "L": "Hangzhou",
      "O": "k8s",
      "OU": "4Paradigm"
    }
  ]
}
EOF
```
* 使用CA签发证书和私钥：`cfssl gencert -ca=/etc/kubernetes/pki/ca.pem -ca-key=/etc/kubernetes/pki/ca-key.pem -config=/etc/kubernetes/pki/ca-config.json -profile=kubernetes metrics-server-csr.json|cfssljson -bare metrics-server`

## apiserver开启聚合配置
>错误：I0313 05:18:36.447202 1 serving.go:273] Generated self-signed cert (apiserver.local.config/certificates/apiserver.crt, apiserver.local.config/certificates/apiserver.key)Error: cluster doesn't provide requestheader-client-ca-file

```
--proxy-client-cert-file=/etc/kubernetes/pki/metrics-server.pem \
--proxy-client-key-file=/etc/kubernetes/pki/metrics-server-key.pem \
--runtime-config=api/all=true \
--requestheader-client-ca-file=/etc/kubernetes/pki/ca.crt \
--requestheader-allowed-names=aggregator \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User
```

## 容器网络配置
>错误【容器解析的主机名不是节点ip】：E0313 08:23:41.193222 1 manager.go:102] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:192.168.209.130: unable to fetch metrics from Kubelet 192.168.209.130 (192.168.209.130)

```
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
```

# prometheus
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
[kubernetes上部署prometheus](https://github.com/coreos/kube-prometheus)

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

# 阿里云产品实践
* 云监控
* SLS：日志服务
* ARMS(java/php)：性能监控
* AHAS(应用高可用服务)：
    - 架构感知
    - 故障演练
    - 流控降级
