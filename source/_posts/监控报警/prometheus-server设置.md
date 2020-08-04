---
title: prometheus-server设置
date: 2020-08-05 01:47:42
tags:
  - prometheus
categories: ['prometheus']
---

# 启动参数

* --config.file=/etc/config/prometheus.yml：指定配置文件
* --storage.tsdb.path=/data：指定数据目录
* --web.enable-lifecycle：允许通过http POST方式关闭服务或重载配置
* --storage.tsdb.retention.time：数据保留时间(默认15天)【y,w, d, h, m, s, ms】
* --storage.tsdb.no-lockfile：不创建lockfile【如果使用k8s的deployment管理时需要开启】

# 主配置

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

# 抓取目标设置-scrape_configs

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

# 目标标签处理-relabel_configs

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

# 规则

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