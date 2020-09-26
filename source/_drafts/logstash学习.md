---
title: logstash学习
tags:
categories:
---

# 工作原理

logstash事件处理管道有三个阶段：

>inputs → filters → outputs

* input：产生事件；使用input将数据获取到Logstash，常见的输入如下，更多[input插件](https://www.elastic.co/guide/en/logstash/6.5/input-plugins.html)
  * file：读取文件，类似unix下的tail -0f命令
  * syslog：从514端口获取syslog信息，并根据RFC3164格式进行解析
  * redis：使用redis管道和redis列表从redis服务器读取数据；redis作为消息服务器，将传入logstash的事件进行排队
  * beats：处理由beats发送的事件
* filter：定义事件；如果事件符合特定条件，则可以将过滤器与条件语句结合使用以对事件执行操作。一些有用的过滤器如下，更多[filter插件](https://www.elastic.co/guide/en/logstash/6.5/filter-plugins.html)
  * grok：解析并构造任意文本。Grok当前是Logstash中将非结构化日志数据解析为结构化和可查询内容的最佳方法。Logstash内置了120种模式，您很可能会找到满足您需求的模式！
  * mutate：对事件字段执行常规转换。您可以重命名，删除，替换和修改事件中的字段。
  * drop：完全删除事件，例如调试事件。
  * clone：复制事件，可能会添加或删除字段。
  * geoip：添加有关IP地址地理位置的信息（还在Kibana中显示惊人的图表！）
* output：将事件传送到任何位置。output是Logstash管道的最后阶段。一个事件可以传递给多个output，只有完成所有output处理，该事件才执行完成。一些有用的output如下，更多[output插件](https://www.elastic.co/guide/en/logstash/6.5/output-plugins.html)
  * elasticsearch：将事件数据发送到Elasticsearch。
  * file：写入事件到文件
  * graphite：将事件数据发送给graphite。*Graphite*是一个开源实时的、显示时间序列度量数据的图形系统。
  * statsd：发送给statsd；该服务“通过UDP侦听统计信息（如计数器和计时器），并将聚合发送到一个或多个可插拔后端服务”。

input和output支持编解码器，使您可以在数据进入或退出管道时对其进行编码或解码，而不必使用单独的过滤器。流行的编解码器包括json、msgpack、plain。更多[codec插件](https://www.elastic.co/guide/en/logstash/6.5/codec-plugins.html)

* json：编解码json格式数据
* multiline：将多行文本事件（例如Java异常和stacktrace消息）合并到单个事件中。

# 使用

* 标准输入/标准输出

```
bin/logstash -e "input{stdin{}}output{stdout{}}"
```

* 自定义格式处理

  ```
  [INFO] 2020-09-26 12:07:15 [cn.itcast.dashboard.Main] - DAU|4903|提交订单|2020-09-26 01:01:14
  [INFO] 2020-09-26 12:07:16 [cn.itcast.dashboard.Main] - DAU|2589|查看订单|2020-09-26 12:05:03
  [INFO] 2020-09-26 12:07:17 [cn.itcast.dashboard.Main] - DAU|8259|使用优惠券|2020-09-26 09:05:13
  ```

  ```
  input {
    beats {
      port => "5044"
    }
  }
  filter {
    mutate {
      split => { "message" => "|" } # 日志切割为多个字段
    }
    mutate {
      add_field => {               # 针对个别字段设置字段名称
        "userID" => "%{message[1]}"
        "visit" => "%{message[2]}"
        "date" => "%{message[3]}"
      }
    }
   mutate {
     convert => {                 # 设置字段类型
        "userID" => "integer"
        "visit" => "string"
        "date" => "string"   
     }
    }
  }
  output {
    elasticsearch {
      hosts => ["127.0.0.1:9200"]
    }
  }
  ```