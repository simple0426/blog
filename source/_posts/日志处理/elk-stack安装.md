---
title: elk-stack安装
tags:
  - elasticsearch
  - logstash
  - filebeat
  - kafka
  - kibana
  - cerebro
  - elasticsearch-head
categories:
  - elk
date: 2020-08-14 17:49:46
---


# 组件介绍

* elasticsearch：基于Lucene的搜索服务器，用于数据收集、分析、存储
* logstash：日志收集、过滤、转换
* kibana：数据可视化
* kafka：数据缓冲队列；提高系统扩展性，可以使关键组件顶住峰值访问压力
* filebeat：使用go编写的收集文件数据的轻量级客户端；隶属于beats系列，其他beats客户端
  * metricbeat：可以采集指标数据，如系统、文件、进程、服务等
  * packetbeat：收集网络数据
  * winlogbeat：收集windows事件日志
  * heartbeat：健康检查

# 架构和数据流

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/elk-kafka.png)

filebeat==》kafka《==logstash==》elasticsearch《==kibana

* filebeat收集数据写入消息队列kafka
* logstash从kafka中读取数据，并写入elasticsearch集群
* kibana从elasticsearch读取数据并在web界面展示

# 安装要求

## 节点环境

| ip/主机名            | 组件                              | 配置   |
| -------------------- | --------------------------------- | ------ |
| 192.168.31.221/elk-1 | es(jdk)/kafka(zookeeper)/cerebro  | >=2C3G |
| 192.168.31.222/elk-2 | es(jdk)/kafka(zookeeper)/kibana   | >=2C3G |
| 192.168.31.223/elk-3 | es(jdk)/kafka(zookeeper)/logstash | >=2C3G |
| 192.168.31.250/-     | filebeat                          | >=2G   |

## 系统设置

* 清空iptables

* 关闭firewalld

  ```
  systemctl stop firewalld 
  systemctl disable firewalld
  ```

* 关闭selinux

  ```
  setenforce 0
  sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config 
  getenforce
  ```

* 主机hosts设置

  ```
  192.168.31.221 elk-1
  192.168.31.222 elk-2
  192.168.31.223 elk-3
  ```

## 软件下载

* jdk【1.8.0_251】：https://www.injdk.cn/
  * checksum：https://www.oracle.com/webfolder/s/digest/8u251checksum.html
* elasticsearch/logstash/kibana/filebeat【6.5.4】：https://mirrors.huaweicloud.com/
* kafka【2.11-2.0.0】：https://mirrors.bfsu.edu.cn/apache/kafka/


# jdk安装

* 解压软件包：tar xzf jdk-8u251-linux-x64.tar.gz -C /usr/local/
* 配置环境变量：source /etc/profile.d/java.sh

```
echo "
JAVA_HOME=/usr/local/jdk1.8.0_251
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME PATH" > /etc/profile.d/java.sh
```

* 测试安装：java -version

# elasticsearch安装

## 安装要求

* 新建普通用户：

  ```
  useradd elastic
  echo "123456"|passwd --stdin elastic
  ```

* 解压软件包：tar xzf elasticsearch-6.5.4.tar.gz -C /usr/local/

* 创建es数据和日志目录

  ```
  mkdir -pv /data/elasticsearch/{data,logs}
  ```

* 修改安装目录和存储目录权限

  ```
  chown -R elastic.elastic /usr/local/elasticsearch-6.5.4/
  chown -R elastic.elastic /data/elasticsearch/
  ```

* 增加最大内存映射和禁用swap【/etc/sysctl.conf】

  ```
  echo "
  vm.swappiness = 0
  vm.max_map_count=655300" >> /etc/sysctl.conf
  sysctl -p
  ```

* 增加最大进程数、最大文件打开数、内存锁定限制【/etc/security/limits.conf】

  ```
  echo "
  * soft nofile 65536
  * hard nofile 65536
  * soft nproc 32000
  * hard nproc 32000
  * hard memlock unlimited
  * soft memlock unlimited
  " >> /etc/security/limits.conf
  ```
  
  实时生效【/etc/systemd/system.conf】
  
  ```
  echo "
  DefaultLimitNOFILE=65536
  DefaultLimitNPROC=32000
  DefaultLimitMEMLOCK=infinity" >> /etc/systemd/system.conf
  systemctl daemon-reload
  ```

## 配置文件

* 主配置【/usr/local/elasticsearch-6.5.4/config/elasticsearch.yml】

  ```
  cluster.name: elk
  node.name: 221
  node.master: true
  node.data: true
  path.data: /data/elasticsearch/data
  path.logs: /data/elasticsearch/logs
  bootstrap.memory_lock: true
  bootstrap.system_call_filter: false
  network.host: 192.168.31.221
  http.port: 9200
  discovery.zen.minimum_master_nodes: 2
  discovery.zen.ping_timeout: 100s
  discovery.zen.fd.ping_timeout: 100s
  discovery.zen.ping.unicast.hosts: ["192.168.31.222","192.168.31.223"]
  discovery.zen.fd.ping_interval: 10s
  discovery.zen.fd.ping_retries: 10
  http.cors.enabled: true
  http.cors.allow-origin: "*"
  ```
  配置文件含义

  ```
  cluster.name: 集群名称【集群中唯一】
  node.name: 节点名称【每个节点都不同】
  node.master: 是否可以成为master
  node.data: 是否为数据节点
  path.data: 数据存储目录
  path.logs: 日志存储
  bootstrap.memory_lock: 内存锁定，是否禁用swap
  bootstrap.system_call_filter: 系统调用过滤器
  network.host: 绑定节点ip【每个节点都不同】
  http.port: rest api地址
  discovery.zen.minimum_master_nodes:集群可正常工作，参与master选举的节点数；官方推荐(N/2)+1，N为集群中所有具有master选择资格的节点数
  discovery.zen.ping_timeout: 单播超时时间
  discovery.zen.fd.ping_timeout: 节点发现超时时间
  discovery.zen.ping.unicast.hosts: 向集群的其他节点发送单播【集群中的其他节点ip】
  discovery.zen.fd.ping_interval: 启动节点发现的时间间隔
  discovery.zen.fd.ping_retries: 节点发现重试次数
  http.cors.enabled: 是否允许跨域访问，可以使用head访问es
  http.cors.allow-origin: 允许的源域
  ```

* jvm设置【/usr/local/elasticsearch-6.5.4/config/jvm.options】

  ```
  -Xms1g #最小内存
  -Xmx1g #最大内存
  ```

  * 堆内存最大和最小值相等，防止程序运行过程中改变堆内存大小
  * 如果系统内存足够大，将最大和最小值设置为31G，因为jvm有32G性能瓶颈
  * 堆内存大小不要超过内存的50%

## 启动-supervisor

* 安装supervisor

  ```
  yum install epel-release -y
  yum install supervisor -y
  systemctl start supervisord
  systemctl enable supervisord
  ```

* 编辑elasticsearch配置：/etc/supervisord.d/es.ini

  ```
  [program:es]
  command=/usr/local/elasticsearch-6.5.4/bin/elasticsearch
  environment=JAVA_HOME="/usr/local/jdk1.8.0_251"
  startsec=30
  priority=10
  directory=/usr/local/elasticsearch-6.5.4/
  autostart=true
  autorestart=true
  user=elastic
  ```

* 启动elasticsearch

  ```
  supervisorctl update
  supervisorctl start elasticsearch
  ```

## cerebro安装

> 可以通过cerebro以web方式访问elasticsearch

* 安装并启动docker

  ```
  sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  sudo yum -y install docker-ce
  vim /etc/docker/daemon.json #docker加速设置
  systemctl start docker
  systemctl enable docker
  ```

* 启动cerebro

  ```
  docker run --name cerebro --restart=always -d -p 9000:9000 lmenezes/cerebro
  ```

* web访问：http://192.168.31.221:9000/

## elasticsearch-head安装

> 可以通过elasticsearch-head以web方式访问elasticsearch

* 安装phantomjs

  ```
  wget https://bitbucket.org/ariya/phantomjs/downloads/phantomjs-2.1.1-linux-x86_64.tar.bz2
  tar xjf phantomjs-2.1.1-linux-x86_64.tar.bz2 -C /usr/local/
  ln -s /usr/local/phantomjs-2.1.1-linux-x86_64/bin/phantomjs /usr/local/bin/
  yum install fontconfig freetype2 -y
  phantomjs -v
  ```

* 安装node

  ```
  wget https://cdn.npm.taobao.org/dist/node/v14.8.0/node-v14.8.0-linux-x64.tar.xz
  tar xJf node-v14.8.0-linux-x64.tar.xz -C /usr/local/
  echo "
  PATH=/usr/local/node-v14.8.0-linux-x64/bin:$PATH
  export PATH" >>  /etc/profile.d/node.sh
  source /etc/profile.d/node.sh
  npm config set registry https://registry.npm.taobao.org
  npm config get registry
  ```

* 安装elasticsearch-head

  ```
  git clone git://github.com/mobz/elasticsearch-head.git
  npm install
  ```

* 修改配置

  * Gruntfile.js添加hostname，此处设置head的访问地址

    ```
        connect: {
                server: {
                        options: {
                                port: 9100,
                                base: '.',
                                keepalive: true,
                                hostname: '*'
                        }
                }
        }
    ```

  * _site/app.js修改es的访问地址

    ```
    this.base_uri = this.config.base_uri || this.prefs.get("app-base_uri") || "http://192.168.31.221:9200";
    ```

* 启动elasticsearch-head

  * 命令行方式

  ```
  npm run start
  ```

  * supervisor方式【npm install -g grunt-cli】

  ```
  [program:es-head]
  command=grunt server
  environment=PATH=/usr/local/node-v14.8.0-linux-x64/bin
  startsec=10
  priority=15
  directory=/usr/local/src/elasticsearch-head
  autostart=true
  autorestart=true
  user=root
  ```

* web访问：http://192.168.31.221:9100

# kibana安装

* 解压安装

  ```
  tar xzf kibana-6.5.4-linux-x86_64.tar.gz -C /usr/local/
  ```

* 配置【config/kibana.yml】

  ```
  server.port: 5601
  server.host: "192.168.31.222"
  elasticsearch.url: "http://192.168.31.221:9200"
  kibana.index: ".kibana"
  ```

* supervisor启动

  ```
  [program:kibana]
  command=/usr/local/kibana-6.5.4-linux-x86_64/bin/kibana
  startsec=30
  priority=20
  directory=/usr/local/kibana-6.5.4-linux-x86_64
  autostart=true
  autorestart=true
  user=root
  ```

* web访问：http://192.168.31.222:5601

* 【可选】使用nginx明文认证方式反向代理kibana

# kafka安装

## 前置要求

* 解压软件包

  ```
  tar xzf kafka_2.11-2.0.0.tgz -C /usr/local/
  ```

* 创建数据和日志目录

  ```
  mkdir -pv /data/zookeeper/{data,logs}
  mkdir -pv /data/kafka/logs
  ```

## 配置文件

* 配置文件修改-zookeeper

  ```
  sed -i 's/^[^#]/#&/' config/zookeeper.properties 
  echo "
  dataDir=/opt/data/zookeeper/data
  dataLogDir=/opt/data/zookeeper/logs
  clientPort=2181
  tickTime=2000
  initLimit=20
  syncLimit=20
  server.1=192.168.31.221:2888:3888
  server.2=192.168.31.222:2888:3888
  server.3=192.168.31.223:2888:3888" >> config/zookeeper.properties
  ```

  配置详情

  ```
  dataDir            zk数据目录
  dataLogDir         zk日志目录
  clientPort         客户端访问zk服务的端口
  tickTime=2000      zk服务器之间或客户端与服务器之间心跳间隔
  initLimit=20       follower（相对于leader而言的客户端）连接到leader进行初始化超时时间，以tickTime为单位
  syncLimit=20       follower与leader同步超时时间
  server.1=192.168.31.221:2888:3888     2888为follower与leader数据交换端口，3888是执行leader选举的通信端口
  server.2=192.168.31.222:2888:3888
  server.3=192.168.31.223:2888:3888
  ```

  为每个kafka节点设置不同id

  ```
  echo 1 > /data/zookeeper/data/myid
  ```

* 修改配置文件-kafka

  ```
  sed -i 's/^[^#]/#&/' config/server.properties
  echo "
  broker.id=1
  listeners=PLAINTEXT://192.168.31.221:9092
  num.network.threads=3
  num.io.threads=8
  socket.send.buffer.bytes=102400
  socket.receive.buffer.bytes=102400
  socket.request.max.bytes=104857600
  log.dirs=/data/kafka/logs
  num.partitions=6
  num.recovery.threads.per.data.dir=1
  offsets.topic.replication.factor=3
  transaction.state.log.replication.factor=3
  transaction.state.log.min.isr=3
  default.replication.factor=3
  log.retention.hours=168
  log.segment.bytes=1073741824
  log.retention.check.interval.ms=300000
  zookeeper.connect=192.168.31.221:2181,192.168.31.222:2181,192.168.31.223:2181
  zookeeper.connection.timeout.ms=6000
  group.initial.rebalance.delay.ms=0" >> config/server.properties
  ```

  配置详情

  ```
  broker.id          每个server需要配置单独的broker id【每个节点不同】，如果不配置系统会自动配置；与上一步骤的id保持一致
  listeners             服务监听地址【每个节点不同】
  num.network.threads   收发信息的线程数
  num.io.threads        处理请求的线程数，包含磁盘IO
  socket.send.buffer.bytes          socket发送缓冲区
  socket.receive.buffer.bytes       socket接收缓冲区
  socket.request.max.bytes          socket接收请求最大字节数
  log.dirs              日志文件目录
  num.partitions        新建Topic时默认的分区数
  num.recovery.threads.per.data.dir          每个数据目录启动时恢复日志、关闭时刷盘日志使用的线程数
  offsets.topic.replication.factor           用于配置偏移量话题的复制因子
  transaction.state.log.replication.factor   事务主题的复制因子
  transaction.state.log.min.isr              覆盖事务主题的min.insync.replicas配置
  default.replication.factor                 自动创建topic时的默认副本的个数 
  log.retention.hours                        日志保留时间
  log.segment.bytes                          单个日志文件大小
  log.retention.check.interval.ms            检查日志是否需要保留的时间间隔
  zookeeper.connect                          zk连接地址
  zookeeper.connection.timeout.ms            zk连接超时
  group.initial.rebalance.delay.ms=0
  ```

## 服务启动

* supervisor启动zookeeper

  ```
  [program:zk]
  command=/usr/local/kafka_2.11-2.0.0/bin/zookeeper-server-start.sh /usr/local/kafka_2.11-2.0.0/config/zookeeper.properties
  environment=JAVA_HOME="/usr/local/jdk1.8.0_251"
  startsec=10
  priority=10
  directory=/usr/local/kafka_2.11-2.0.0
  autostart=true
  autorestart=true
  user=root
  ```

  使用nc验证zk启动

  ```
  yum install nc -y
  echo conf|nc 192.168.31.221 2181 #配置
  echo stat|nc 192.168.31.221 2181 #集群状态
  ```

* supervisor启动kafka

  ```
  [program:kafka]
  command=/usr/local/kafka_2.11-2.0.0/bin/kafka-server-start.sh /usr/local/kafka_2.11-2.0.0/config/server.properties
  environment=JAVA_HOME="/usr/local/jdk1.8.0_251"
  startsec=10
  priority=15
  directory=/usr/local/kafka_2.11-2.0.0
  autostart=true
  autorestart=true
  user=root
  ```

## 测试

环境变量设置

  ```
echo -e "PATH=/usr/local/kafka_2.11-2.0.0/bin/:\$PATH\nexport PATH" > /etc/profile.d/kafka.sh
source /etc/profile
  ```

  创建topic

  ```
kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic testtopic 
kafka-topics.sh --zookeeper localhost:2181 --list 
  ```

  模拟生产者和消费者【不同终端】

  ```
kafka-console-producer.sh --broker-list 192.168.31.221:9092 --topic testtopic
kafka-console-consumer.sh --bootstrap-server 192.168.31.222:9092 --topic testtopic --from-beginning
  ```

## web管理工具

kafka-manager：https://github.com/yahoo/CMAK

# logstash安装

* 安装要求

  ```
  tar xzf logstash-6.5.4.tar.gz -C /usr/local/
  ```

* 配置文件

  ```
  input{
  kafka{
    type => "nginx_kafka"
    codec => "json"
    topics => "nginx"
    decorate_events => true
    bootstrap_servers => "192.168.31.221:9092,192.168.31.222:9092,192.168.31.223:9092"
  }
  }
  output{
    if[type] == "nginx_kafka"{
      elasticsearch {
        hosts => ["192.168.31.221","192.168.31.222","192.168.31.223"]
        index => "logstash-nginx-%{+YYYY-MM-dd}"
     }
    }
  }
  ```

  配置详解

  ```
  input{                       //数据来源
  kafka{                       //来源于kafka
    type => "nginx_kafka"     //自定义的数据分类
    codec => "json"           //解码数据格式
    topics => "nginx"          //从kafka的哪个topic获取数据
    decorate_events => true    //可向事件添加Kafka元数据，比如主题、消息大小的选项
    bootstrap_servers => "192.168.31.221:9092,192.168.31.222:9092,192.168.31.223:9092" //kafka服务地址
  }
  }
  output{                         //数据流向
    if[type] == "nginx_kafka"{    //数据分类判断
      elasticsearch {             //数据写入es
        hosts => ["192.168.31.221","192.168.31.222","192.168.31.223"]            //es服务地址
        index => "logstash-nginx-%{+YYYY-MM-dd}"                  //向es写入数据时创建的索引
     }
    }
  }
  ```

* supervisor启动

  ```
  [program:logstash]
  command=/usr/local/logstash-6.5.4/bin/logstash -r -f /usr/local/logstash-6.5.4/config/logstash.conf
  environment=JAVA_HOME="/usr/local/jdk1.8.0_251"
  startsec=10
  priority=20
  directory=/usr/local/logstash-6.5.4
  autostart=true
  autorestart=true
  user=root
  ```

* 测试安装结果

  ```
  kafka-topics.sh --zookeeper localhost:2181 --list #可以看到nginx这个topic
  ```

# filebeat安装

* 安装要求

  ```
  tar xzf filebeat-6.5.4-linux-x86_64.tar.gz -C /usr/local/
  ```

* 配置文件

  ```
  filebeat.prospectors:
  - input_type: log
    paths:
      - /var/log/nginx/access.log
    json.keys_under_root: true
    json.add_error_key: true
    json.message_key: log
  output.kafka:
    hosts: ["192.168.31.221:9092","192.168.31.222:9092","192.168.31.223:9092"]
    topic: 'nginx'
  ```

  配置详解

  ```
  filebeat.prospectors:
  - input_type: log                      # 输入数据类型，日志文件log，标准输入stdin
    paths:                               # 日志文件路径
      - /var/log/nginx/access.log
    json.keys_under_root: true           # 让字段位于根节点
    json.add_error_key: true             # 将解析错误的消息存储在error.message字段
    json.message_key: log                # 合并多行json日志，需要配置multiline
  output.kafka:                          # 输出到kafka
    hosts: ["192.168.31.221:9092","192.168.31.222:9092","192.168.31.223:9092"]
    topic: 'nginx'
  ```

* supervisor启动

  ```
  [program:filebeat]
  command=/usr/local/filebeat-6.5.4-linux-x86_64/filebeat -e -c /usr/local/filebeat-6.5.4-linux-x86_64/filebeat.yml
  startsec=10
  priority=25
  directory=/usr/local/filebeat-6.5.4-linux-x86_64
  autostart=true
  autorestart=true
  user=root
  ```

# 测试与查看

* 访问nginx
* kibana创建索引模式、查看访问日志：http://192.168.31.222:5601/