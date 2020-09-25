---
title: kibana学习
tags:
categories:
---

# docker方式启动

```
docker run -d --name kibana --link elastic:elasticsearch -p 8601:5601 elastic/kibana:6.5.4
```

Docker默认选项

| `server.name`                                   | `kibana`                    |
| ----------------------------------------------- | --------------------------- |
| `server.host`                                   | `"0"`                       |
| `elasticsearch.hosts`                           | `http://elasticsearch:9200` |
| `monitoring.ui.container.elasticsearch.enabled` | `true`                      |

环境变量设置

| **Environment Variable** | **Kibana Setting**   |
| ------------------------ | -------------------- |
| `SERVER_NAME`            | `server.name`        |
| `SERVER_BASEPATH`        | `server.basePath`    |
| `MONITORING_ENABLED`     | `monitoring.enabled` |

# 页面功能

```
Discovery         发现：索引数据搜索
Visualize         可视化：根据索引数据创建可视化界面
Dashboard         仪表盘：将多个可视化界面聚合为一个仪表盘
Dev tools         开发工具：可用于调试es语法、logstash的grok语法
Management        管理：索引管理
```

# 内置范例

* 安装nginx软件
* 使用filebeat的nginx modules抓取nginx日志并格式化
* filebeat对接elasticsearch，并在elasticsearch中自动创建索引filebeat-\*
* 使用filebeat的kibana设置，在kibana中创建dashboards（默认使用索引filebeat-\*）

* 在kibana中搜索es索引
  * 在discovery中查看文本型nginx日志
  * 在dashboard中查看图表型nginx访问记录

# 自定义功能

* 在Visualize中创建单个可视化图表并保存
  * 设置Y轴，一般为数据
  * 设置X轴，一般是时间

* 在dashboard最终创建仪表盘并保存
  * 在仪表盘中添加可视化图表并调整位置