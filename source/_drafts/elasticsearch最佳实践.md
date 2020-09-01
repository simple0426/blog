---
title: elasticsearch最佳实践
tags:
categories:
---

# [容量评估](https://help.aliyun.com/document_detail/72660.html)

## 磁盘总容量

* 原始数据
  * 每日原始数据(GB)
  * 每日增量数据
  * 保留时间
* 预留资源
  * 设置分片设置多少副本：至少1个
  * 索引开销：通常比源数据大10%
  * 操作系统预留：5%
  * Elasticsearch内部开销：段合并、日志等内部操作，预留20%。
  * 安全阈值：通常至少预留15%的安全阈值。

原始数据 = 每日原始数据 \* 保留天数 \* 膨胀系数 

磁盘总存储 = 原始数据  \*  (1 + 副本数) \*（1 + 索引开销）\* /（1 - Linux预留空间）/（1 - Elasticsearch开销）/（1 - 安全阈值）

                    = 原始数据 \*（1 + 副本数）\* 1.7
    
                    =  原始数据 \* 3.4

## 分片

* 数据量和索引数
  * 创建多少索引(定期轮转、删除、合并)
  * 每个索引的数据量
  * 每个索引配置的主分片和副本分片
* 分片大小：
  * 单个分片大小保持在10GB - 50GB之间(30GB最优);日志分析或者超大索引场景，建议单个shard大小不要超过100GB。
  * 分片过大：可能使 ES 的故障恢复速度变慢
  * 分片过小：可能导致非常多的分片，但因为每个分片使用一些数量的 CPU 和内存，从而导致读写性能、内存不足等问题。
* 分片数量：
  * 单节点分片数量(限额) = 当前节点的内存大小 \* 30(TB级别以下)；在单节点上，7.x版本的实例默认的shard的上限为1000个
  * 单索引分片数量(包括副本数)设置为接近数据节点的整数倍，方便分片在所有数据节点均匀分布。
  * 当数据量很大时：选择多主1副本，降低Elasticsearch压力
  * 当数据量较小(低于30GB)：使用单主多副本(负载均衡，提高性能)或者单主1副本。

## 节点规格

- 集群最大节点数：集群最大节点数 = 单节点CPU数 * 5。

- 单节点磁盘最大容量：

  使用场景不同，单节点最大承载数据量也会不同，具体如下：

  - 数据加速、查询聚合等场景：单节点磁盘最大容量 = 单节点内存大小（GB）* 10
  - 日志写入、离线分析等场景：单节点磁盘最大容量 = 单节点内存大小（GB）* 50
  - 通常情况：单节点磁盘最大容量 = 单节点内存大小（GB）* 30

# 集群

## [节点](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/modules-node.html)

* master节点：控制集群状态；node.master设置为true，节点具有成为master的资格【默认true】
* data节点：执行数据CRUD、搜索、聚合操作的节点；设置参数：node.data【默认true】
* ingest节点：在数据创建索引前对数据进行处理，代替部分logstash功能；设置参数：node.ingest【默认true】
* 协调节点：在执行搜索、批量索引过程中，每个数据节点将本地执行的结果汇总到协调节点(接受客户端请求的节点)，协调节点将数据执行汇总、聚合等操作后返回给客户端
  * 每个节点都是一个隐含的协调节点，都可以接受客户端请求
  * node.master、node.data、node.ingest都设置为false时，节点是一个专用协调节点
* 部落节点：特殊类型的协调节点，可以连接不同集群，执行跨集群搜索；tribe.*

## 集群状态

* 接口：_cluster/health
* 状态：
  * green：所有主分片和副本分片都可用
  * yellow：主分片都可用，副本分片部分可用
  * red：主分片部分可用

## 分片和副本

* 分片(shard)：将索引的数据分解为大小合适的片段(分片)分散在不同节点进行存储
  * 分片可以提升数据读写速度、扩充集群存储容量
  * 每一个分片都是一个Lucene实例，都是一个完整的搜索引擎，应用程序不会和它直接通信。
  * 每个文档都属于一个单独的主分片
  * 索引建立后分片数不可调整，主分片的数量决定了索引能够存储多少数据
* 副本：主分片的完全复制品，它和主分片内容完全一样
  * 保证了分片的高可用
  * 也可以响应读请求
  * 索引建立后，副本可以调整

## 路由

确定文档的存储和读取位置

shard=hash(routing) % number_of_primary_shards

* routing是一个随机字符串，默认为_id，可以自定义
* 这个routing字符串通过hash函数生成一个数字，然后除以主分片数得到一个余数，余数的范围永远是0到number_of_primary_shards - 1,这个数字就是特定文档所在的分片【这也决定了主分片一旦建立就不能修改：增加主分片会造成无法定位之前的文档位置】

# ingest操作

## 创建管道流
```
curl -XPUT http://47.93.186.106:9200/_ingest/pipeline/my-pipeline-id -d '
{
    "description":"describe pipeline",
    "processors":[
        {
            "convert": {
                "field": "foo",
                "type": "integer"
            }
        }
    ]
}
'
```
## 使用管道流发送数据
```
curl -XPOST http://47.93.186.106:9200/filebeat-2018.01.08/testlog?pipeline=my-pipeline-id -d '
{
    "date": "1534966686000",
    "user": "hejingqi",
    "mesg": "first message into Elasticsearch",
    "foo": "12"
}
'
```
## 搜索插入结果
```
curl http://47.93.186.106:9200/filebeat-2018.01.08/testlog/_search?q=hejingqi
```

# 索引管理curator
* python包elasticsearch-curator
* 存储优化
* 删除旧索引

# 集群性能测试esrally
* python3.4包esrally
* 参考：<https://github.com/elastic/rally>

# ansible管理es
* 参考：<https://github.com/elastic/ansible-elasticsearch>

# 报警插件elastalert
* 参考：<http://elastalert.readthedocs.io/en/latest/elastalert.html#>

# web管理工具
* bigdesk
* cerebro