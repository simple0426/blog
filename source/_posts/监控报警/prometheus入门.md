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


# 架构与组件

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/prometheus-architecture.png)

* 数据采集(pull模型)：
  * 短周期任务(short-lived jobs)：通过__pushgateway__组件短期存储指标数据
  * 持续性任务：exporter
  * 服务发现(service discovery)：自动发现监控指标，例如：文件、k8s、consul
* 数据存储(__prometheus server__)：node-hdd/ssd(本地存储)、influxDB
* 告警(__altermanager__)
* 查询&展示：promQL(查询语言)、api clients、__grafana__、web UI

# prometheus-server

## 启动参数

* --config.file=/etc/config/prometheus.yml：指定配置文件
* --storage.tsdb.path=/data：指定数据目录
* --web.enable-lifecycle：允许通过http POST方式关闭服务或重载配置
* --storage.tsdb.retention.time：数据保留时间(默认15天)【y,w, d, h, m, s, ms】
* --storage.tsdb.no-lockfile：不创建lockfile【如果使用k8s的deployment管理时需要开启】

## 主配置

* 主配置示例：https://github.com/prometheus/prometheus/blob/release-2.20/config/testdata/conf.good.yml
* 抓取设置和标签设置示例：https://github.com/prometheus/prometheus/blob/release-2.20/documentation/examples/prometheus-kubernetes.yml
* 主配置

```yaml
global:
  # 收集数据时间间隔(default = 1m)
  scrape_interval: 30s

  # 收集数据超时(default = 10s)
  scrape_timeout: 15s

  # 评估规则时间间隔(default = 1m)
  #evaluation_interval: 30s

  # 和外部通信(联邦、远程存储、报警)时添加的到任意时间序列或警报的标签
  #external_labels:
  #  [ <labelname>: <labelvalue> ... ]

  # PromQL查询日志，重载配置文件会重新打开一个文件
  #query_log_file: query.log

# 报警规则文件，可以使用正则表达式进行文件匹配
rule_files:
#- "first.rules"
#- "my/*.rules"

# 抓取数据的配置
scrape_configs:
  # 默认添加标签`job=<job_name>`到时序数据
  - job_name: 'prometheus'
    # metrics_path 数据指标节点url，默认'/metrics'
    # scheme 使用的协议，默认'http'.

    static_configs:
    - targets: ['localhost:9090']
# 指定与Alertmanager相关的配置
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# 可以将数据写入其他时序数据库【如influxDB】
#remote_write:
#  - url: "http://localhost:8086/api/v1/prom/write?db=prometheus&u=username&p=password"

# 可以读取其他时序数据库【如influxDB】
#remote_read:
#  - url: "http://localhost:8086/api/v1/prom/read?db=prometheus&u=username&p=password"
```

## 抓取目标设置-scrape_configs

```yaml
scrape_configs: #抓取设置
- job_name: prometheus #分组设置，这个分组下的所有目标会被添加job=prometheus的标签
  static_configs:   # 配置文件中设置抓取目标
  - targets: ['127.0.0.1:9090'] #抓取目标，同时也会在这个目标上添加标签instance=127.0.0.1:9090
    labels:  #从此数据源获取的数据都会添加标签my=label
      my: label
- job_name: node_export
  file_sd_configs: # 加载配置文件(yaml/json)获取抓取目标
    - files: 
      - foo/*.slow.json
      - foo/*.slow.yml
      - single/file.yml
      refresh_interval: 10m #读取配置文件时间间隔【默认5min】
    - files:
      - bar/*.yaml
- job_name: service-y
  consul_sd_configs: #通过配置中心consul获取抓取目标
  - server: 'localhost:1234'
    token: mysecret
    services: ['nginx', 'cache', 'mysql'] #只抓取匹配的服务
    tags: ["canary", "v1"]
    node_meta:
      rack: "123"
    allow_stale: true
    scheme: https
    tls_config:
      ca_file: valid_ca_file
      cert_file: valid_cert_file
      key_file:  valid_key_file
      insecure_skip_verify: false
```

* 目标配置文件(json格式)

```
[
  {
    "targets": [ "<host>", ... ],
    "labels": {
      "<labelname>": "<labelvalue>", ...
    }
  },
  ...
]
```

## 标签处理-relabel_configs

> 可以直接在数据抓取目标处设置label

* drop、keep：删除或保留正则匹配的原标签

  ```
    - action: keep #只保留原标签中值为default、kubernetes、https的标签
      regex: default;kubernetes;https
      source_labels:
      - __meta_kubernetes_namespace
      - __meta_kubernetes_service_name
      - __meta_kubernetes_endpoint_port_name
  ```

* replace：替换标签key或value

  * 替换标签内容【使用replacement】

  ```
    - action: replace #执行替换操作
      regex: ([^:]+)(?::\d+)?;(\d+) #对原标签的value进行匹配(分号分隔匹配不同标签的正则)
      replacement: $1:$2 #目标标签value(可以使用正则的分组引用原标签的value)
      source_labels: #原标签key
      - __address__
      - __meta_kubernetes_service_annotation_prometheus_io_port
      target_label: __address__ #目标标签key
    # 此例子中使用正则匹配(regex)从原标签(source_labels)
    # 的值中提取值(replacement)赋值给目标标签(target_label)
  ```

  * 替换标签(重命名标签)

  ```
    - action: replace
      source_labels:
      - __meta_kubernetes_service_name
      target_label: kubernetes_name
    # 将原标签(source_labels)重命名为目标标签
  ```

* labelmap：只保留标签名称的部分内容

  ```
    - action: labelmap
      regex: __meta_kubernetes_service_label_(.+)
    # 如果标签名称为__meta_kubernetes_service_label_asdfg，则被简化为asdfg
  ```

## 规则

* 记录规则【对原始的数据进行计算分类，避免使用过多数据的计算冲击prometheus的运行】

  ```
  groups:
    - name: example
      rules:
      - record: job:http_inprogress_requests:sum
        expr: sum by (job) (http_inprogress_requests)
  ```

* 报警规则

  ```
  groups:
  - name: node.rules #分组名称
    rules:
    - alert: NodeFilesystemUsage #告警名称
      expr: | #表达式
        100 - (node_filesystem_free{fstype=~"ext4|xfs"} / 
        node_filesystem_size{fstype=~"ext4|xfs"} * 100) > 20 
      for: 1m #持续时间
      labels: #标签【过滤选择资源】
        severity: warning  #报警级别
      annotations: #注解，告警详情
        summary: "Instance {{ $labels.instance }} : {{ $labels.mountpoint }} 分区使用率过高"
        description: "{{ $labels.instance }}: {{ $labels.mountpoint }} 分区使用大于80% (当前值: {{ $value }})"
  ```

# pushgateway

* prometheus配置抓取目标为pushgateway时，必须设置honor_labels为true，避免收集数据本身的job和instance被覆盖
* 配置参数【--persistence.file】和--persistence.interval=5m将收集的数据持久化

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
  * 内置函数：rate(区间内每秒增长量)、irate、abs、increase(区间内增长量)、sort_desc、sort

# altermanager-告警

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
