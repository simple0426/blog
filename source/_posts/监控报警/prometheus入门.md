---
title: prometheus入门
tags:
  - prometheus
  - altermanager
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

# 核心概念-数据

## 数据格式(data model)

* 语法：`<metric name>{<label name>=<label value>, ...}`
* 范例：`api_http_requests_total{method="POST", handler="/messages"}`
* 数据产生流程
  * 客户端metrics接口的数据格式：【node_boot_time_seconds 1.585383507e+09】
  * prometheus接收数据后根据数据来源添加标签后进行存储【node_boot_time_seconds{instance="localhost:9100",job="node_exporter"}  1585383507】

## 数据类型(metric types)

* 计数器类型(counter)：计数器是一个累计度量指标，

  * 代表一个单调递增的计数器(除非系统重启)
  * 例如代表请求数、任务完成数、错误数

* 测量类型(gauge)：反应应用的当前状态

  * 数据可增可减
  * 例如服务器的内存、cpu、磁盘使用率等

* 直方图类型(Histogram)：显示一段时间内值在各区间的分布情况【概率分布】

  * 数据类型名称：度量基础名称+不同类型统计数据

    * bucket类型：获取值在统计区间(bucket)的个数，名称如<basename>_bucket{le="<upper inclusive bound>"}
    * sum类型：所有获取数值的总和，名称如<basename>_sum
    * count类型：所有获取值的个数，名称如<basename>_count【等同于<basename>_bucket{le="+Inf"}（小于无穷大的个数）】

  * 范例

    ```
    prometheus_http_response_size_bytes_bucket{handler="/metrics",le="100"} 0 #http响应小于1000字节的个数
    prometheus_http_response_size_bytes_bucket{handler="/metrics",le="1000"} 0 
    prometheus_http_response_size_bytes_bucket{handler="/metrics",le="10000"} 163
    prometheus_http_response_size_bytes_bucket{handler="/metrics",le="100000"} 170
    prometheus_http_response_size_bytes_bucket{handler="/metrics",le="1e+06"} 170
    prometheus_http_response_size_bytes_bucket{handler="/metrics",le="1e+07"} 170
    prometheus_http_response_size_bytes_bucket{handler="/metrics",le="1e+08"} 170
    prometheus_http_response_size_bytes_bucket{handler="/metrics",le="1e+09"} 170
    prometheus_http_response_size_bytes_bucket{handler="/metrics",le="+Inf"} 170
    prometheus_http_response_size_bytes_sum{handler="/metrics"} 1.578861e+06 # #170次http响应的字节总和
    prometheus_http_response_size_bytes_count{handler="/metrics"} 170 #一共抓取170次http响应数据
    ```

  * 数据处理：histogram_quantile(a float, b instant-vector)函数可计算区间b中超过分数a的个体情况；例如：统计http请求超过10分钟的90%场景中的情况(剔除极端值)：histogram_quantile(0.9, rate(http_request_duration_seconds_bucket[10m]))

* 摘要类型(Summary)：和直方图数据类似，它是由客户端直接提供(histogram_quantile)的计算结果提供给prometheus

  ```
  prometheus_target_interval_length_seconds{interval="15s",quantile="0.01"} 14.999831807
  prometheus_target_interval_length_seconds{interval="15s",quantile="0.05"} 14.999910011
  prometheus_target_interval_length_seconds{interval="15s",quantile="0.5"} 15.000031073
  prometheus_target_interval_length_seconds{interval="15s",quantile="0.9"} 15.000114733
  prometheus_target_interval_length_seconds{interval="15s",quantile="0.99"} 15.000228675
  prometheus_target_interval_length_seconds_sum{interval="15s"} 10170.020818119998
  prometheus_target_interval_length_seconds_count{interval="15s"} 678
  ```

## 作业和实例(jobs and instances)

* instance：可以抓取数据的目标端点

* jobs：具有相同目标的实例(instance)集合【如多个mysql实例】

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


# 架构与组件

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/prometheus-architecture.png)

* 数据采集(pull模型)：
  * 短周期任务(short-lived jobs)：通过__pushgateway__组件短期存储指标数据
  * 持续性任务：exporter
  * 服务发现(service discovery)：自动发现监控指标，例如：文件、k8s、consul
* 数据存储(__prometheus server__)：node-hdd/ssd(本地存储)、influxDB
* 告警(__altermanager__)
* 查询&展示：promQL(查询语言)、api clients、__grafana__、web UI

# PromQL语法

> 学习方法：通过granfa的[dashboard](https://grafana.com/grafana/dashboards)模板

* 瞬时数据：node_cpu_seconds_total
* 区间数据：node_load1[3m]
  * `s` - seconds
  * `m` - minutes
  * `h` - hours
  * `d` - days
  * `w` - weeks
  * `y` - years
* 标签选择指标：node_network_info{operstate="up",device=~"veth.*"}
  * =：选择与字符串完全相等的标签
  * !=：选择与字符串不相等的标签
  * =~：选择与字符串正则匹配的标签
  * !~：选择与字符串正则不匹配的标签

* 指标计算：(node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes)*100（可用内存）
  * 加减乘除：+、-、*、/、%、^
  * 比较运算(常用于报警)：==、!=、>、>=、<、<=
  * 逻辑运算：and、or
  * 聚合运算：sum、min、max、avg、count、topk(n,metric)
    * sum：
      * 直接计算总和(所有节点可用内存)：sum(node_memory_MemAvailable/(1024*1024))
      * 分类之后分别计算总和(计算每个pod的占用内存)：sum(container_memory_rss{image!=""}) by(pod) 
  * 内置函数：rate(区间内每秒增长量)、irate、abs、increase(区间内增长量)、sort_desc、sort

# pushgateway

* prometheus配置抓取目标为pushgateway时，必须设置honor_labels为true，避免收集的数据本身的job和instance被覆盖
* 配置参数【--persistence.file】和--persistence.interval=5m将收集的数据持久化

# 未完待续

* service discovery

* 自定义监控指标-API

* prometheus-operator

*  [高可用prometheus：thanos 实践](https://segmentfault.com/a/1190000022164038)