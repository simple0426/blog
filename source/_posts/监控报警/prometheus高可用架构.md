---
title: prometheus高可用架构
date: 2020-08-05 01:41:33
tags:
  - prometheus
  - altermanager
categories: ['prometheus']
---

# prometheus高可用

## 联邦功能

prometheus原生支持联邦架构，能够实现从别的prometheus抓取符合特定条件的数据

```
scrape_configs:
- job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job=~"kubernetes.*"}' #抓取目标prometheus中job为kubernetes开头的监控项
    static_configs:
      - targets:
        - '192.168.31.201:30090'
```

## 高可用方案

* HA方案：启动多个prometheus-server，每个prometheus配置都相同，收集相同目标的数据

  ![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/prometheus-ha.jpg)

  * 优点：只能保证prometheus服务的可用性问题
  * 缺点：数据无法持久化、无法保证数据一致性；受制于单点性能限制(ServerA/ServerB)，无法进行动态扩展。
  * 适用场景：小规模、短周期存储监控数据

* HA+远程存储：多台配置相同的prometheus(ServerA/ServerB)，都可以向远程存储写数据；使用keepalived机制，保证同一时间只启动其中的一台prometheus-server，也只有存活的prometheus可以向远程存储写数据

  ![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/prometheus-remote-storage.jpg)

  * 优点：数据可以长期存储、保证了数据的一致性；保证了prometheus-server的可用性
  * 缺点：单台server性能有限，无法收集更多监控数据
  * 适用场景：小规模、长期数据存储

* HA+远程存储+联邦集群：将不同的采集任务划分为不同功能区(ServerA/ServerB)，再启动一台单独的prometheus(ServerC)将重要的数据从ServerA和ServerB中汇总，同时启用外部存储进行数据持久化

  ![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/prometheus-ha-federate.jpg)

  * 优点：重要数据可以持久化、server可以灵活迁移；能够依据不同任务进行层级划分；ServerA/ServerB可以使用HA进行高可用

## 监控范例

* 监控需求：
  * 系统监控(cpu、内存、磁盘、网络)数据量不大，但需要长期存储【需要做资源规划和分析】
  * 业务监控(nginx/grpc等)和线上业务访问成正比，数据量巨大；业务监控主要做实时探测，一般需求不超过一周(主要做实时业务成功率报警，历史数据分析从ELK等日志系统操作)

* 监控实现

  ![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/prometheus-sample.jpg)

# altermanager高可用

单点架构中，由于收敛规则中的group可以去重(repeat)，所以altermanager可以对接多个具有相同配置的prometheus-server

## 高可用方案

* 使用负载均衡，将prometheus发送的告警分流到不同的altermanager上

  ![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/alert-ha.jpg)

* 使用altermanager的集群功能【使用gossip协议：去中心化、最终一致性】

  * --cluster.listen-address="127.0.0.1:8001"：当前实例集群服务监听地址
  * --cluster.peer=127.0.0.1:8001：初始化时关联的其他实例的集群服务地址【第一个启动的实例不用配置】

## 集群实践

* webhook项目编译

  * webhook项目：https://github.com/prometheus/alertmanager/tree/master/examples/webhook

  * 安装go环境：https://golang.google.cn/doc/install

  * 配置go仓库：

    ```
    go env -w GO111MODULE=on
    go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct
    ```

  * 编译webhook项目：`go get github.com/prometheus/alertmanager/examples/webhook`

  * webhook最新二进制文件位置：$HOME/go/bin/webhook

* 启动altermanager和webhook

  * [alertmanager.yml](https://github.com/prometheus/alertmanager/blob/master/examples/ha/alertmanager.yml)：关联本地webhook

  ```
  nohup ./alertmanager  --web.listen-address=":9093" --cluster.listen-address="127.0.0.1:8001" --config.file=alertmanager.yml --log.level=debug 2>&1 > alert1.log & 
  nohup ./alertmanager  --web.listen-address=":9094" --cluster.listen-address="127.0.0.1:8002" --cluster.peer=127.0.0.1:8001 --config.file=alertmanager.yml --log.level=debug 2>&1 > alert2.log & 
  nohup ./alertmanager  --web.listen-address=":9095" --cluster.listen-address="127.0.0.1:8003" --cluster.peer=127.0.0.1:8001 --config.file=alertmanager.yml --log.level=debug 2>&1 > alert3.log & 
  nohup ./webhook --log.level=debug 2>&1 > webhook.log &
  ```

* 使用脚本测试最终收到的报警信息
  * 报警脚本：https://github.com/prometheus/alertmanager/blob/master/examples/ha/send_alerts.sh
  * 查看报警信息：`cat webhook.log`