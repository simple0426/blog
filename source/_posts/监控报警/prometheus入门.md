---
title: prometheus入门
tags:
  - prometheus
categories:
  - prometheus
date: 2020-07-30 21:41:36
---


# 主要特点

* 一个多维数据模型(data model)，其中包含通过度量名称(metric name)和标签(key/value)定义的时间序列数据
* PromQL，灵活的数据查询语言
* 不依赖分布式存储，数据保存在服务器节点上
* 通过http的pull模型拉取时间序列数据
* 可以通过推送网关(pushgateway)支持push模型获取数据
* 可以通过服务发现或静态配置发现目标节点
* 支持多种图形和仪表盘

# 核心概念

* 数据模型(data model)

  * 语法：`<metric name>{<label name>=<label value>, ...}`
  * 范例：`api_http_requests_total{method="POST", handler="/messages"}`

* 度量类型(metric types)

* 作业和实例(jobs and instances)

  * instance：可以抓取数据的目标端点

  * jobs：具有相同目标的实例(instance)集合

  * 范例：

    ```
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
    - job_name: 'node'
      static_configs:
      - targets: ['192.168.1.10:9090']
    ```

    

# 架构

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/prometheus-architecture.png)

* 数据采集(pull模型)：
  * 短周期任务(short-lived jobs)：通过pushgateway短期存储指标数据
  * 持续性任务：exporter
  * 服务发现(service discovery)：自动发现监控指标，例如：文件系统、k8s、consul
* 数据存储：node-hdd/ssd(本地存储)、influxDB
* 告警：altermanager
* 查询&展示：promQL(查询语言)、api clients、grafana、web UI

# k8s监控

## 监控指标

* node节点状态、资源利用率
* 资源对象(serivce/deployment/pod)状态、数量
* 容器资源利用率
* 应用程序
  * 程序性能数据，例如java项目jvm exporter
  * 业务数据

## 客户端组件

* cadvisor(kubelet)：采集容器cpu、内存使用情况；PromQL查询关键词：`container_`
* node_exporter：采集linux节点cpu、内存使用情况；PromQL查询关键词：`node_`
* kube-state-metrics：通过API Server收集资源对象的状态并生成相关的指标；PromQL查询关键词：`kube_`

## 部署

* 方式1：[kube-prometheus](https://github.com/coreos/kube-prometheus)【coreos】
* 方式2：单独部署各组件
  * prometheus
    * [配置文件](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#alertmanager_config)
  * grafana
    * dashboard模板：https://grafana.com/grafana/dashboards
  * 客户端组件【默认既有cadvisor】
    * node_exporter
    * kube-state-metrics
  * altertmanager【在prometheus中配置告警规则】

# 告警

## 告警规则

    groups:
    - name: node.rules #分组名称
      rules:
      - alert: NodeFilesystemUsage #告警名称
        expr: | #表达式
          100 - (node_filesystem_free{fstype=~"ext4|xfs"} / 
          node_filesystem_size{fstype=~"ext4|xfs"} * 100) > 20 
        for: 1m #持续时间
        labels: #标签【过滤选择资源】
          severity: warning 
        annotations: #注解，告警详情
          summary: "Instance {{ $labels.instance }} : {{ $labels.mountpoint }} 分区使用率过高"
          description: "{{ $labels.instance }}: {{ $labels.mountpoint }} 分区使用大于80% (当前值: {{ $value }})"
## 告警状态

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/alter-status.jpg)

* inactive：当做什么都没发生
* pending：已触发阈值，但是为满足告警持续时间
* firing：触发阈值且满足告警持续时间；告警移送给接收者。

## [告警收敛设置](https://prometheus.io/docs/alerting/latest/alertmanager/#alertmanager)

* 分组(group)：将告警名称相同的多个告警合并到一个告警中发送
* 抑制(inhibit)：当告警发生后，停止发送和此类警报相关的其他警报
* 静默(silence)：是一种简单的在特定时间范围内将警报静音的机制

## 告警流程

* prometheus根据采集的数据和告警规则进行判断；满足条件后，将告警发送到altermanager
  * 是否触发阈值
  * 是否超过持续时间
* altermanager根据告警收敛规则，决定是否、什么时间发送告警（邮件、微信、钉钉等）

