---
title: elk-安全和访问控制
date: 2020-11-01 22:10:42
tags: ['xpack', '安全']
categories: ['elk']
---

本文主要基于[通过 TLS 加密和基于角色的访问控制确保 Elasticsearch 的安全](https://www.elastic.co/cn/blog/getting-started-with-elasticsearch-security)完成

# 前言

从elastic stack 6.8和7.1开始，会在软件分发版本中免费提供多项安全功能，例如elasticsearch之间的TLS加密通信，kibana中基于角色的访问控制(RBAC)等.

在完成下列操作前，应当完成以下操作：

* 创建一个3节点的elasticsearch集群
* elasticsearch集群安装并使用elasticsearch-head插件
* 已安装filebeat、logstash、kibana等组件

# ES间TLS通信设置

在es的配置文件elasticsearch.yml的同级目录生成证书

```
bin/elasticsearch-certutil cert -out config/elastic-certificates.p12 -pass ""
```

修改配置文件elasticsearch.yml

```
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

将证书复制到其他节点相应位置、修改配置文件elasticsearch.yml

重启所有节点elasticsearch服务

# ES设置密码

```
bin/elasticsearch-setup-passwords interactive
```

可以在任意节点，使用上述命令交互式的设置用户密码，交互过程中会产生如下用户：

* elastic：超级用户
* kibana：kibana连接es用户
* logstash_system：logstash连接es用户
* filebeat_system：filebeat连接es用户
* apm_system
* remote_monitoring_user

为了操作方便，可以将如上用户密码设置一致

# ES客户端密码访问设置

## logstash设置

```
output {
 elasticsearch {
   user => "elastic"
   password => "123456"
 }
}
```

设置范例如上，设置后重启logstash；

如果使用logstash_system会出现如下错误，解决方案未知，暂时使用超级用户连接

```
Oct 31 23:40:21 elk-2 logstash: [2020-10-31T23:40:21,881][ERROR][logstash.outputs.elasticsearch] Encountered a retryable error. Will Retry with exponential backoff  {:code=>403, :url=>"http://192.168.31.222:9200/_bulk"}
```

## kibana设置

```
elasticsearch.username: "kibana"
elasticsearch.password: "123456"
```

设置后重启kibana，此时kibana即可连接elasticsearch；

当web登录kibana时使用超级管理员(即elastic/123456)进行认证

## elasticsearch-head设置

* 修改elasticsearch.yml

  ```
  http.cors.enabled: true
  http.cors.allow-origin: "*"
  http.cors.allow-headers: Authorization,X-Requested-With,Content-Length,Content-Type
  ```

* 使用如下连接方式登录elasticsearch-head

  ```
  http://{host}:9100/?auth_user=elastic&auth_password=123456
  ```

## API访问设置

* curl方式

  ```
  curl -u username:password
  ```

* postman方式

  ```
  authorization ==> Basic Auth ==> Username/Password
  ```

* python api

  ```
  from elasticsearch import Elasticsearch
  
  node_list = [
      {"host": "192.168.31.221", "port": 9200},
      {"host": "192.168.31.222", "port": 9200},
      {"host": "192.168.31.223", "port": 9200}
  ]
  es = Elasticsearch(hosts=node_list, http_auth=('elastic', '123456'))
  ```

# kibana中配置基于角色的访问控制(RBAC)

* 创建角色(roles)
  * 选择索引模式名称(例如：logstash-nginx-access*)
  * 选择可以对索引的权限(例如：read)

* 创建用户
  * 设置用户名和密码
  * 选择角色名称【需要为用户额外分配**kibana_user**角色(因为该用户将会查看 Kibana 中的数据)】

使用创建的用户登录，即可管理(查看)相应的索引数据