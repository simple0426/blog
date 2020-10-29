---
title: filebeat学习
tags:
  - filebeat
categories:
  - elk
date: 2020-09-24 01:54:54
---


# 工作原理

filebeat由2个组件组成：inputs和harvester 

* harvester：负责单个文件的读取；如果文件在读取时被删除或重命名，filebeat依然可以读取文件【harvester运行时占用一个文件描述符保证相关的文件依然被保留】

* inputs：负责管理harvester并查找所有可以读取的资源；如果输入类型是日志，则查找器查找路径匹配的所有文件，并为每个文件启动一个harvester；支持多种inputs类型，例如log、stdin、tcp、udp、docker、redis、syslog等

filebeat如何保持文件状态？

Filebeat会保留每个文件的状态，并经常将状态刷新到注册表文件(registry)中的磁盘。该状态用于记住收割机正在读取的最后一个偏移量，并确保发送所有日志行。如果无法到达输出（例如Elasticsearch或Logstash），Filebeat会跟踪发送的最后几行，并在输出再次可用时继续读取文件。当Filebeat运行时，状态信息也保存在内存中，用于每个输入。重新启动Filebeat时，将使用注册表文件中的数据来重建状态，并且Filebeat会将每个收割机继续到最后一个已知位置。

对于每个输入，Filebeat会保持找到的每个文件的状态。由于可以重命名或移动文件，因此文件名和路径不足以标识文件。对于每个文件，Filebeat都存储唯一的标识符以检测文件是否以前被收获过。

注册表文件位置：

```
filebeat.registry_file: ${path.data}/registry
```

# 模块

可以将常见的日志进行格式化，直接输出到elasticsearch或kibana【此时就不需要再设置filebeat.inputs】

* 在elasticsearch中安装ingest-geoip和ingest-user-agent

> 安装要求：https://www.elastic.co/guide/en/beats/filebeat/6.5/filebeat-modules-quickstart.html
>
> 其他版本说明：[v6.7之后作为elasticsearch的ingest processor无需安装](https://www.elastic.co/guide/en/elasticsearch/plugins/6.7/ingest-geoip.html)
>
> docker方式：docker启动的elasticsearch已默认安装

```
sudo bin/elasticsearch-plugin install ingest-geoip
sudo bin/elasticsearch-plugin install ingest-user-agent
```

* 启停模块

```
./filebeat modules enable nginx
./filebeat modules list
./filebeat modules disable nginx
```

* 模块配置参数【modules.d/nginx.yml】

```
- module: nginx
  access:
    enabled: true
    var.paths: ["/var/log/nginx/access.log*"] #设置nginx日志位置
  error:
    enabled: true
    var.paths: ["/var/log/nginx/error.log*"]
```

* filebeat的主配置

```
filebeat.config.modules:
  reload.enabled: true                 # 运行动态加载配置
  path: ${path.config}/modules.d/*.yml # 加载的模块位置
```

# 命令行工具

```
Usage:
  filebeat [flags]
  filebeat [command]

命令选项：
export   # 导出配置
modules  # 模块管理
test     # 测试配置
setup    # 在相关环境中进行初始化设置
* --dashboards      在kibana中创建dashboard【kibana可用】
* --machine-learnin 设置机器学习的job配置【机器学习节点可用】
* --pipelines       在es中创建pipeline【es的ingest节点可用】
* --template        在es中设置索引mappings模板

参数：
-c filebeat.yml  # 指定配置文件名称(默认filebeat.yml)，相对于path.config
-d publish       # 指定字符串调试
-e               # filebeat日志记录到标准输出
--path.data      # 数据目录       
--path.home      # 根目录
--path.logs      # 日志目录

启动范例：
./filebeat -e -c filebeat.yml -d "publish"
```

# [配置选项](https://www.elastic.co/guide/en/beats/filebeat/6.5/configuring-howto-filebeat.html)

## inputs设置

```
filebeat.inputs:
- type: log                  # 输入数据类型，日志文件-log
  enabled: true              # 开启log类型功能,默认true
  paths:                     # 日志文件路径,使用通配符
  - /usr/local/filebeat-6.5.4/*.log
  exclude_files: ['.gz$']    # 排除指定的文件
  include_lines: ["^ERR", "^WARN"] # 只发送包含这些字样的日志【默认包含所有；在exclude_lines之前调用】
  exclude_lines: ["^OK"]     # 不发送包含这些字样的日志
  tags: ["nginx", "web"]     # 添加标签
  fields:                    # 添加字段，默认添加到子节点fields下
    env: staging
  fields_under_root: true    # 添加字段到根节点下
  ignore_older: "24h"        # 超过 24小时没更新内容的文件不再监听。
  scan_frequency: "10s"      # 每 10秒钟扫描一次目录，更新通配符匹配上的文件列表
  tail_files: false          # 是否从文件末尾开始读取
  harvester_buffer_size: 16384 # 实际读取文件时，每次读取 16384 字节
  backoff: "1s"              # 每 1 秒检测一次文件是否有新的一行内容需要读取
```

## [多行日志-multiline](https://blog.csdn.net/UbuntuTouch/article/details/106272704)

* multiline.pattern：指定要匹配的正则表达式模式。
* multiline.negate：定义是否为否定模式，也就是和上面定义的模式相反。 默认为false。
* multiline.match：如何将模式匹配和不匹配的行组合到一个事件中

| pattern：^b 匹配以b开始的行                            | negate | match  | 结果                                                         |
| ------------------------------------------------------ | ------ | ------ | ------------------------------------------------------------ |
| ![](https://img-blog.csdnimg.cn/20200522095441298.png) | false  | after  | 模式匹配的连续行追加到不匹配的行之后<br>【1-bb行追加到a行之后，2-bb行追加到c行之后】 |
| ![](https://img-blog.csdnimg.cn/20200522095832263.png) | false  | before | 模式匹配的连续行添加到下一个不匹配的行之前<br>【1-bb行添加到a行之前，2-bb行追加到c行之前】 |
| ![](https://img-blog.csdnimg.cn/20200522100310568.png) | true   | after  | 模式不匹配的连续行添加到匹配行之后<br>【ac行追加到1-b行之后，de行追加到2-b行之后】 |
| ![](https://img-blog.csdnimg.cn/20200522100724855.png) | true   | before | 模式不匹配的连续行添加到下一个匹配行之前<br>【ac行添加到1-b行之前，de行添加到2-b行之前】 |

* multiline.max_lines：单个事件中可以包含的最大行数，默认500

* multiline.timeout：单个事件的读取超时时间，默认5s

* multiline.flush_pattern：可以指定单个事件的结束匹配模式，这对于事件以特定标记开始和结束有用

  ```
  [2015-08-24 11:49:14,389] Start new event
  [2015-08-24 11:49:14,395] Content of processing something
  [2015-08-24 11:49:14,399] End event
  ```

  ```
  multiline.pattern: ‘Start new event’
  multiline.negate: true
  multiline.match: after
  multiline.flush_pattern: ‘End event’
  ```

​       当看到“Start new event”模式并且以下几行与该模式不匹配时，它们将被追加到与该模式匹配的前一行。 
​       当看到以“End event”开头的一行时，flush_pattern选项将指示多行事件结束。

* 范例1

```
Exception in thread "main" java.lang.NullPointerException
        at com.example.myproject.Book.getTitle(Book.java:16)
        at com.example.myproject.Author.getBookTitles(Author.java:25)
        at com.example.myproject.Bootstrap.main(Bootstrap.java:14)
```

```
multiline.pattern: ^[[:space:]] # 匹配空格开始的行
multiline.negate: false         # 连续多行与匹配模式相同
multiline.match: after          # 将匹配的行添加到不匹配的行之后
```

* 范例2

```
[beat-logstash-some-name-832-2015.11.28] IndexNotFoundException[no such index]
    at org.elasticsearch.cluster.metadata.IndexNameExpressionResolver$WildcardExpressionResolver.resolve(IndexNameExpressionResolver.java:566)
    at org.elasticsearch.cluster.metadata.IndexNameExpressionResolver.concreteIndices(IndexNameExpressionResolver.java:133)
    at org.elasticsearch.cluster.metadata.IndexNameExpressionResolver.concreteIndices(IndexNameExpressionResolver.java:77)
    at org.elasticsearch.action.admin.indices.delete.TransportDeleteIndexAction.checkBlock(TransportDeleteIndexAction.java:75)
```

```
multiline.pattern: ^\[   # 匹配以[开始的行
multiline.negate: true   # 连续多行与匹配模式相反
multiline.match: after   # 将不匹配的行放在匹配行之后
```

* 范例3

```
Exception in thread "main" java.lang.IllegalStateException: A book has a null property
       at com.example.myproject.Author.getBookIds(Author.java:38)
       at com.example.myproject.Bootstrap.main(Bootstrap.java:14)
Caused by: java.lang.NullPointerException
       at com.example.myproject.Book.getId(Book.java:22)
       at com.example.myproject.Author.getBookIds(Author.java:35)
       ... 1 more
```

```
multiline.pattern: '^[[:space:]]+(at|\.{3})[[:space:]]+\b|^Caused by:' # 匹配2个条件之一，条件如下
multiline.negate: false # 连续多行与匹配模式相同
multiline.match: after  # 将匹配的行放到不匹配的行之后
```

> 以空白开始，接着是 at 或3个点，再一个空白之后接任意字符
>
> 以 Caused by: 开始

## 内部队列

Filebeat在发布事件之前使用内部队列存储事件。

```
queue.mem:
  events: 4096          # 队列可以存储的事件数，默认4096
  flush.min_events: 512 # 发布所需的最小事件数。如果将此值设置为0，则输出可以开始发布事件，而无需额外的等待时间。否则，输出必须等待更多事件可用。默认2048
  flush.timeout: 5s    # 发布需要等待的最大等待时间flush.min_events。如果设置为0，事件将立即可供使用。默认1s
```

## output设置

- [Elasticsearch](https://www.elastic.co/guide/en/beats/filebeat/6.5/elasticsearch-output.html)

  ```
  output.elasticsearch:
    hosts: ["https://localhost:9200"]
    index: "filebeat-%{[beat.version]}-%{+yyyy.MM.dd}"
    username: "filebeat_internal"
    password: "YOUR_PASSWORD"
    ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]
    ssl.certificate: "/etc/pki/client/cert.pem"
    ssl.key: "/etc/pki/client/cert.key"
  ```

- [Logstash](https://www.elastic.co/guide/en/beats/filebeat/6.5/logstash-output.html)

  ```
  output.logstash:
    hosts: ["127.0.0.1:5044"]
  ```

- [Kafka](https://www.elastic.co/guide/en/beats/filebeat/6.5/kafka-output.html)

  ```
  output.kafka:
    hosts: ["kafka1:9092", "kafka2:9092", "kafka3:9092"]
    topic: '%{[fields.log_topic]}'  # 读取inputs中设置的自定义字段fields.log_topic的值
  ```

- [Redis](https://www.elastic.co/guide/en/beats/filebeat/6.5/redis-output.html)

- [File](https://www.elastic.co/guide/en/beats/filebeat/6.5/file-output.html)

- [Console](https://www.elastic.co/guide/en/beats/filebeat/6.5/console-output.html)

  ```
  output.console:
    pretty: true
    enable: true
  ```

### 负载均衡output主机

```
output.logstash:
  hosts: ["localhost:5044", "localhost:5045"]
  loadbalance: true
```

## elasticsearch模板

filebeat可以在elasticsearch中创建索引，相关设置：**setup.template.\***

开启方式：

* 命令行方式：filebeat setup --template
* 配置文件方式：setup.template.enabled: true【默认true】

其他模板设置选项：

```
setup.template.settings:
  index.number_of_shards: 1  
  index.number_of_replicas: 1 
```

> 根据模板创建的默认索引名称(按天分隔)："filebeat-%{[beat.version]}-*"

## kibana仪表盘

filebeat(beats系列)可以在kibana中创建dashboard，相关设置：**setup.dashboards-\***、**setup.kibana**

开启方式：

* 命令行方式：filebeat setup --dashboards
* 配置文件方式：setup.dashboards.enabled: false【默认false】

其他模板设置选项：


```
setup.dashboards.enabled: true              # filebeat启动，在kibana创建filebeat类的dashboard
setup.kibana:                               # kibana连接地址
  host: "localhost:8601"
```

> filebeat在kibana中创建的dashboard默认引用的索引名称是：filebeat-\*

## filebeat日志

```
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

