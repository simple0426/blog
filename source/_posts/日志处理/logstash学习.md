---
title: logstash学习
tags:
  - logstash
categories:
  - elk
date: 2020-10-29 01:13:38
---


# [配置文件语法](https://www.elastic.co/guide/en/logstash/6.8/configuration.html)

## 基础语法

* 区块标识 `{}`
* 变量赋值 `nginx => 5`
* 变量引用 `[nginx]`
  * 双引号下引用变量 `"%{nginx}"`
  * 引用数组 `[arry][1]`

## 变量数据类型

* 数字
* 字符串
* 布尔值
* 数组【列表】

```
match => ["datetime", "UNIX", "ISO8601"]
```

* 哈希【字典】

```
options => {
  key1 => "value1",
  key2 => "value2"
}
```

## 条件判断

```
if EXPRESSION {
  ...
} else if EXPRESSION {
  ...
} else {
  ...
}
```

* ==（等于）!=（不等于） < <=（小于，小于等于）>>=（大于，大于等于）
* =~（匹配正则）!~（不匹配正则）
* in（包含）not in（不包含）
* and（与）or（或）nand（非与）xor（非或）
* （）（复合表达式）!()（复合表达式取反）

# 插件

logstash事件处理管道有三个阶段：inputs → filters → outputs。input和output都支持编解码器。常见的几种插件类型如下：

* [Input](https://www.elastic.co/guide/en/logstash/6.8/input-plugins.html)：产生事件；使用input将数据获取到Logstash
* [Filter](https://www.elastic.co/guide/en/logstash/6.8/filter-plugins.html)：定义事件；如果事件符合特定条件，则可以将过滤器与条件语句结合使用以对事件执行操作。
* [Output](https://www.elastic.co/guide/en/logstash/6.8/output-plugins.html)：将事件传送到任何位置。output是Logstash管道的最后阶段。一个事件可以传递给多个output，只有完成所有output处理，该事件才执行完成。
* [Codec](https://www.elastic.co/guide/en/logstash/6.8/codec-plugins.html)：编解码器

插件管理命令如下：

* bin/logstash-plugin install         # 安装
* bin/logstash-plugin remove      # 删除
* bin/logstash-plugin update      # 更新
* bin/logstash-plugin list             # 列表显示

## input

input类插件包含一些通用配置项：add_field(增加的字段)、codec(编解码器)、tags(标签)、type(类型)

* stdin：标准输入，主要用于调试

  ```
  input {
    stdin{}
  }
  ```

* file：读取文件，类似unix下的tail -0f命令

  ```
  file {
    path => "/var/log/messages"
    type => "syslog"
    tags => ["syslog", "test"]
    start_position => "beginning"
  }
  ```

* beats：处理由beats发送的事件

  ```
  input {
    beats {
      port => 5044
    }
  }
  ```

* redis：从redis中读取数据，redis一般作为filebeat的数据缓冲服务

  ```ruby
  input {
    redis {
      host => "127.0.0.1"
      port => "6379"
      password => "123456"
      key => "logstash"
      data_type => "list"
    }
  }
  ```

## filter

filter插件包含一些通用选项：add_field、add_tag、remove_field、remove_tag

* mutate：对事件字段执行常规转换。您可以重命名，删除，替换和修改事件中的字段。

    ```
  [INFO] 2020-09-26 12:07:15 [cn.itcast.dashboard.Main] - DAU|4903|提交订单|2020-09-26 01:01:14
  [INFO] 2020-09-26 12:07:16 [cn.itcast.dashboard.Main] - DAU|2589|查看订单|2020-09-26 12:05:03
  [INFO] 2020-09-26 12:07:17 [cn.itcast.dashboard.Main] - DAU|8259|使用优惠券|2020-09-26 09:05:13
  ```

    ```
  filter {
    mutate {
      split => { "message" => "|" } # 内容分割
      add_field => {                # 添加字段
        "userID" => "%{message[1]}"
        "visit" => "%{message[2]}"
        "date" => "%{message[3]}"
       }
      convert => {                  # 转换数据类型
        "userID" => "integer"
        "visit" => "string"
        "date" => "string"   
      }
    }
  }
    ```

* geoip：将ip地址解析为相应的经纬度、国家、城市等信息

  ```
  filter {
    geoip {
     source => "message"
    }
  }
  ```

* dissect：可以处理有明显分隔符的文本；不使用正则表达式解析，速度也更快

  ```
  # Dec 14 10:35:01 opserver12 CRON[25115]: (root) CMD (command -v debian-sa1 > /dev/null && debian-sa1 1 1)
  dissect {
      mapping => {
         "message" => "%{ts} %{+ts/2} %{+ts/1} %{src} %{prog}[%{pid}]: %{msg}"
      }
     convert_datatype => {
         pid => "int"
     }
  }
  
  * %{key}    表示字段
  * %{+key/n} n表示聚合字段附加的第几次内容【原始字段%{key}等同%{+key/0}】
  * convert_datatype：转换输出数据类型
  ```

  ```
  # http://www.baidu.com/test?he=123
  dissect {
      mapping => {
          "message" => "http://%{domain}/%{?url}?%{?arg1}=%{&arg1}"
      }
  }
  
  * %{?string} 表示只是占位符，不会生产捕获字段最终的event【不会显示字段】
  * %{?string} %{&string} 表示这是一个键值对
  ```

* date：处理日期文本

  ```
  filter {
    date {
     locale => "en_US" # 语言环境
     match => ["message", "dd/MM/yyyy:HH:mm:ss", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss", "ISO8601"] # 匹配文本
     timezone => "Asia/Shanghai" # 文本表示的时区
     target => "logdate" # 将文本赋值给字段
    }
  }
  ```

* grok：通过正则表达式匹配任意文本

## output

* stdout：标准输出，主要用于调试

* file：写入事件到文件

* elasticsearch：将事件数据发送到Elasticsearch。

  ```
  output {
   elasticsearch {
     hosts => ["192.168.31.221:9200","192.168.31.222:9200","192.168.31.223:9200"]
     index => "logstash-%{type}-%{+YYYY.MM.dd}"
   }
  }
  ```
  
  索引名称因为时区滞后8小时处理【新建日期字段】
  
  ```
  filter {
      ruby {
          code => "event.set('index_date', event.get('@timestamp').time.localtime + 8*60*60)"
      }
      mutate {
          convert => ["index_date", "string"]
          gsub => ["index_date", "T([\S\s]*?)Z", ""]
          gsub => ["index_date", "-", "."]
      }
  }
  output {
   elasticsearch {
     hosts => ["192.168.31.221:9200","192.168.31.222:9200","192.168.31.223:9200"]
     index => "logstash-%{type}-%{index_date}"
   }
  }
  ```

## codec

* json：编解码json格式数据

* multiline：将多行文本事件（例如Java异常和stacktrace消息）合并到单个事件中。

  ```
input {
    stdin {
      codec => multiline {
        pattern => "^\s" # 以空白开始的行合并到上边非空白开始的行
        what => "previous"
      }
    }
  }
  ```

# grok插件

## 使用方式

可以使用正则表达式匹配任意文本，根据正则表达式位置的不同可以分为：

* 实时解析模式：在match中定义正则表达式

  ```
  (?<field_name>the pattern here) 
  ```

* 预编译模式：单独定义模式后，在match中引用模式名称

  ```
  %{PATTERN_NAME:field_name:data_type}
  ```

预编译模式可以直接使用内置的模式，也可以自定义模式：

* 内置模式：约有120种，通过[logstash-patterns-core](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns)插件实现
* 自定义模式：可以在任意目录的任意文件里定义，只需通过patterns_dir参数加载目录即可使用

## 模式使用

预编译模式使用步骤：【在线调试工具：http://grokdebug.herokuapp.com/ 】

* 模式的定义：`PATTERN_NAME  pattern`（内置模式无需定义，可以直接使用；）

    * 正则表达式默认不匹配换行，如果需要，则在正则前加(?m)

    * 语法范例：
    
        ```
        USERNAME [a-z0-9.-]+    定义模式USERNAME是由小写字母数字等组成
        NGINXLOG %{COMBINEDAPACHELOG} (%{QS:proxyip}|-) (%{BASE16FLOAT:upstream_response_time}|-) (%{BASE16FLOAT:request_time})   使用内置的模式组合定义
        ```

* 模式的应用：%{PATTERN_NAME:field_name:data_type} 

    * PATTERN_NAME：引用的模式名称
    * field_name：模式匹配的文本输出到指定字段
    * data_type：定义输出字段数据类型

## 多模式匹配

```
# patterns/custom
CUSTOM_DATE [0-9]{1,4}\-[0-9]{1,2}\-[0-9]{1,2} [0-9]{1,2}\:[0-9]{1,2}\:[0-9]{1,2}
```

```
filter {
  grok {
    patterns_dir => "./patterns"
    match => { # 字典方式
      "message" => [
         "%{CUSTOM_DATE:date}", # 如果是日期类型使用date字段存储数据
         "%{WORD:any}"          # 如果是其他文字使用any字段存储
         ]
      }
   # match => ["message", "%{CUSTOM_DATE:date}", "%{WORD:any}"] 列表方式
  }
}
```

# 命令行启动

* -f 指定配置文件路径
* -e 指定配置字符串
* -t 测试配置语法并退出
* -r 自动加载配置更新

```
bin/logstash -r -f grok.rb
```

# [综合范例](https://www.elastic.co/guide/en/logstash/6.8/config-examples.html)

```
NGINXLOG %{IPORHOST:remote_addr} - %{USER:remote_user} \[%{HTTPDATE:time_local}\] \"%{WORD:method} %{URIPATH:baseurl}(?:\?%{NOTSPACE:param}|) HTTP/%{NUMBER:httpversion}\" %{NUMBER:http_status} %{NUMBER:body_bytes_sent} \"%{GREEDYDATA:http_referer}\" \"%{GREEDYDATA:http_user_agent}\" \"(?<http_x_forwarded_for>.*?)\"
```

```
input {
   file {
    type => "nginx-access"  
    path => [ "/home/zj-ops/test/input/logs/access.log" ]
    tags => [ "nginx","access"]
    start_position => beginning
   }
    file {
    type => "nginx-error" 
    path => [ "/home/zj-ops/test/input/logs/error.log" ]
    tags => [ "nginx","error"]
    start_position => beginning
}
}
filter {
 if [type] == "nginx-access" {
        grok{
            patterns_dir => "./patterns"
            match => ["message","%{NGINXLOG}"] # 预编译模式
        }
        date{
            match => ["time_local","dd/MMM/yyyy:HH:mm:ss Z"]
            target => "logdate"
        }
        ruby{
            code => "event.set('logdateunix',event.get('logdate').to_i)"
        }
    } else if [type] == "nginx-error" { 
        grok {
        match => [
            "message", "(?<time>\d{4}/\d{2}/\d{2}\s{1,}\d{2}:\d{2}:\d{2})\s{1,}\[%{DATA:err_severity}\]\s{1,}%{GREEDYDATA:err_message}"] #实时解析模式
        }
        date{
            match => ["time_local","dd/MMM/yyyy HH:mm:ss Z"]
            target => "logdate"
        }
        ruby{
            code => "event.set('logdateunix',event.get('logdate').to_i)"
        }
    }
}
output{
   elasticsearch{
        hosts => ["10.10.10.10:9200"]
        index => "logstash-nginx-%{+YYYY.MM.dd}"
    }
}
```

