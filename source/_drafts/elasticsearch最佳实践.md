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

# ingest

* 在数据创建索引前对数据进行处理，代替部分logstash功能
* 设置elasticsearch.yml【默认开启】：【node.ingest: true】

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