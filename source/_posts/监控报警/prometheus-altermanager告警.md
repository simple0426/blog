---
title: prometheus-altermanager告警
date: 2020-08-05 01:52:23
tags:
  - alertmanager
categories: ['prometheus']
---

# 告警状态

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/alter-status.jpg)

* inactive：当做什么都没发生
* pending：已触发阈值，但是未满足告警持续时间
* firing：触发阈值且满足告警持续时间；告警发给接收者。

# [告警收敛设置](https://prometheus.io/docs/alerting/latest/alertmanager/#alertmanager)

* 分组(group)：将告警名称相同的多个告警合并到一个告警中发送
* 抑制(inhibit)：当告警发生后，停止发送和此类警报相关的其他警报
* 静默(silence)：是一种简单的在特定时间范围内将警报静音的机制

# 告警流程

* prometheus根据采集的数据和告警规则进行判断；满足条件后，将告警发送到altermanager
  * 是否触发阈值
  * 是否超过持续时间
* altermanager根据告警收敛规则，决定是否、什么时间发送告警（邮件、企业微信、钉钉等）

# 启动配置-告警方式

* 启动参数
  * 配置文件：--config.file="alertmanager.yml"
  * 数据目录：--storage.path="data/"
  * 外部访问url：--web.external-url=/
* 主配置【邮件示例】

```
global: 
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.163.com:25'
  smtp_from: 'baojingtongzhi@163.com'
  smtp_auth_username: 'baojingtongzhi@163.com'
  smtp_auth_password: 'NCKBJTSASSXMRQBM'

receivers:
- name: default-receiver
  email_configs:
  - to: "zhenliang369@163.com"

route:
  group_interval: 1m
  group_wait: 10s
  receiver: default-receiver
  repeat_interval: 1m
```