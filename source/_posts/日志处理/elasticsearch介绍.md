---
title: elasticsearch介绍
tags:
  - elasticsearch
  - search
categories:
  - elk
date: 2020-08-25 11:09:25
---

# 引言

## 数据分类和搜索

* 结构化数据：数据有固定格式和长度，在数据存储前定义表结构；通过对常用的字段建立索引进行搜索
* 非结构化数据：没有预定义的结构化特征(比如：文章、网页、邮件等)；需要对全文数据建立倒排索引进行搜索【全文搜索】

## 索引分类

* 索引：对于关系型数据库，一般是针对字段名称建立索引，然后索引指向一条数据记录

* 倒排索引：是针对文档(字段)内容建立索引，然后索引指向文档(数据记录)，实现了term到doc list的映射

# 基本概念

* 索引(index)：由分片构成的逻辑集合，也是文档在物理上的容器，相当于mysql的库；此处索引就是倒排索引，准确的说是一组倒排索引，即多个【term与doc list】的映射
* 映射(mapping)和类型(type)：
  * 映射：是将文档转变为索引的过程，类似于定义mysql的表结构，也是文档进行结构化存储的过程
  * 类型：是不同的映射分类(相当于mysql中的不同表)，是文档的逻辑分类；
    * v6.0之前：一个索引可以有多个类型
    * v6.0-v7.0：一个索引只能有一个类型
    * v7.0之后：删除类型定义，直接映射中定义数据类型【默认创建为\_doc类型，文档操作也是在\_doc类型/接口下完成】
* 分片(shard)：将索引的数据分解为大小合适的片段(分片)分散在不同节点进行存储；这样可以提升数据读写速度、扩充集群存储容量
* 副本(replicas)：主分片的完全复制品，它和主分片内容完全一样；保证了分片的高可用，同时也可以响应读请求
* 文档(doc)：一条记录，是es中数据存储和检索的基本单位；相当于mysql的一行记录；以json格式存储，由元数据(metadata)和数据(_source)组成；元数据包括：
  * _index：文档所在索引
  * _type：文档类型，每个类型都有自己的映射(mapping)
  * _id：文档id
  * _version：文档版本
* 字段(field)：一个文档的数据(_source)由若干个字段组成，每个字段可以有不同的数据类型
  * 基本类型：比如字符串、数值、日期、布尔、字节、范围
  * 扩展类型：数组(列表)、对象(字典)
  * 多数据类型：让一个字段同时具备多种数据类型特征
* 项(term)：索引的最小单位
  * 某个field对应的内容如果是text类型，会将内容进行分词为若干term
  * 如果是不分词的字段(如keyword类型)，那么该字段的内容就是一个term

## [节点](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/modules-node.html)

* master节点：控制集群状态；node.master设置为true，节点具有成为master的资格【默认true】
* data节点：执行数据CRUD、搜索、聚合操作的节点；设置参数：node.data【默认true】
* ingest节点：在数据创建索引前对数据进行处理，代替部分logstash功能；设置参数：node.ingest【默认true】
* 协调节点：接收客户端搜索、批量索引等请求，将请求转发到相应的数据节点执行；最后，将来自数据节点的执行结果合并后返回给客户端
  * 每个节点都是一个隐含的协调节点，都可以接受客户端请求
  * node.master、node.data、node.ingest都设置为false时，节点是一个专用协调节点
* 部落节点：特殊类型的协调节点，可以连接不同集群，执行跨集群搜索；tribe.*

## 分片(shard)

分片(shard)：将索引的数据分解为大小合适的片段(分片)分散在不同节点进行存储

* 分片可以提升数据读写速度、扩充集群存储容量
* 每一个分片都是一个Lucene实例，都是一个完整的搜索引擎，应用程序不会和它直接通信。
* 每个文档都属于一个单独的主分片
* 索引建立后分片数不可调整，主分片的数量决定了索引能够存储多少数据

## 副本(replicas)

副本：主分片的完全复制品，它和主分片内容完全一样

* 保证了分片的高可用
* 也可以响应读请求，多个分片以轮询的方式响应请求
* 索引建立后，副本数可以调整【应用程序刚接入集群时，可以先关闭副本功能，减轻集群负载压力】

## 路由

读写操作时，确定文档所在主分片的位置

shard=hash(routing) % number_of_primary_shards

* routing是一个随机字符串，默认为文档_id，可以自定义
* 这个routing字符串通过hash函数生成一个数字，然后除以主分片数得到一个余数，余数的范围永远是0到number_of_primary_shards - 1,这个数字就是特定文档所在的分片【这也决定了主分片数一旦确定就不能修改：增加主分片会造成无法定位之前的文档位置】

# [存储原理](https://www.cnblogs.com/huangying2124/p/12230177.html)

## 倒排索引

es存储文档也就是建立倒排索引，流程如下：

* 将文档中的关键字(term)提取出来，即文档分析(analysis)
* 将关键字与文档的对应关系保存起来
* 对关键字本身做索引排序

### 基本结构

倒排索引包含以下几个部分

* 某个关键词的doc list
* 某个关键词在所有doc的数量：IDF（inverse document frequency）
* 某个关键词在每个doc中出现的次数：TF（term frequency）
* 某个关键词在这个doc中的次序
* 某个doc的长度
* 某个关键词在所有doc的平均长度

记录这些信息，就是为了方便搜索和_score分值计算

### 不可变性

倒排索引写入磁盘后就是不可变的，这样由几个好处：

* 索引不需要更新，可以不需要锁，可以支持高并发
* 如果cache内存足够，索引不更新，索引可以一直保存在os cache中，可以提高IO性能
* 如果数据不变，filter cache会一直驻留在内存中
* 索引数据可以压缩，从而可以节省cpu、内存、磁盘等资源

## 文档底层原理

### 文档写入

* 新文档先写入索引的内存buffer中；为了安全，也一起写到translog

* 内存buffer的数据每隔一段时间(默认1s)写入index segment中(即编入索引)，并写入os cache中【此时文档就可以搜索】

  * index segment数据写入os cache中，并打开搜索的过程叫refresh，默认1s一次

  * 如果希望该文档能立刻被搜索，可以手动调用refresh操作

  * 日志量较大，但实时性要求不高的系统，可以增大refresh参数

    ```
    {
      "settings": {
        "refresh_interval": "30s" 
      }
    }
    ```

* 等待操作系统将os cache中的数据刷新到磁盘中

* 数据刷新到磁盘后，buffer数据被清空，同时打开一个新的index segment

### 文档修改和删除

被修改和删除的文档，会被标记为deleted，信息记录在.del文件中

修改文档时会创建新文档【segment是不可变内容】

被标记为deleted的文档不会被物理删除，也会被搜索到，但是在最终结果处理时会被过滤掉

执行segment合并时，被标记为deleted的文档会被物理删除掉

### translog

translog是一种日志预写机制，在对数据操作的同时，也将对数据的增删改操作记录在日志文件中，防止数据在持久化之前丢失；translog的相关设置如下：

* translog写磁盘方式(参数：index.translog.durability)：定义了在每次index、delete、update、bulk操作后是否提交并刷新translog到磁盘；包含：同步(request)、异步(async)

  * request(默认)：每次请求，都提交并刷新translog到磁盘；主分片和副本分片都会触发，保障了数据安全性，但是有性能损失；设置方式如下：

    ```
    {
      "settings": {
         "index.translog.durability": "request"
      }
    }
    ```

  * async：不管写操作如何，仅在_index.translog.sync_interval_时间间隔后将translog刷新到磁盘【默认5s，不能小于100ms】；设置方式如下：

    ```
    {
      "settings": {
         "index.translog.durability": "async",
         "index.translog.sync_interval": "5s"
      }
    }
    ```

* 定义flush操作：flush会将内存中的数据(segment和translog)都刷新到磁盘中；产生新的translog，旧的translog被归档

  * 参数定义：index.translog.flush_threshold_size：translog大小达到多少时，执行flush操作

  * 手动执行flush操作

    ```
    POST twitter/_flush
    ```

* translog文件设置
  * index.translog.retention.size：保留的translog文件总大小，默认512mb；保留更多的translog，可以使用本地translog进行故障恢复，而非基于文件同步进行
  * index.translog.retention.age：translog最长保留时间，默认12h

### segment合并

当索引数据不断增长时，对应的segment也会不断的增多(默认，每秒执行一次refresh，产生一个新的segment文件)，查询性能也会下降。因此，Elasticsearch会触发segment合并的线程，把很多小的segment合并成更大的segment，然后删除小的segment。   

标记为deleted的文档在合并时会被丢弃（delete请求只是将文档标记为deleted状态，真正的物理删除是在段合并的过程中）

手动执行segment合并：【参数：max_num_segments：最终合并为几个segment】

```
POST /twitter/_forcemerge
{
  "max_num_segments": 1
}
```

# 分布式原理

## 文档写操作

> 新建、索引、删除文档

* 客户端发送请求到任意节点(协调节点)，该节点根据文档id确定文档的主分片位置，并将请求转发到主分片所在的节点
* 在主分片执行写操作完成后，将相关操作复制到所有副本分片上
* 副本分片执行完成后向主分片报告成功状态，主分片再向协调节点报告成功状态，最后协调节点向客户端报告成功状态

## 文档读操作

> 搜索(单个文档)、查询文档

读操作和写操作流程类似，但是为了负载均衡读请求，协调节点会将读请求以循环的方式发送给所有的副本分片(包含主分片)

可能的情况是，文档已存在主分片上，但还未同步到副本分片上，(假如请求转发到此分片上)副本分片会向协调节点报告文档未找到，由主分片返回文档信息。

## 全文搜索

* 分散(scatter阶段)
  * 客户端发送搜索请求到协调节点，协调节点创建一个from+size的空队列
  * 协调节点将搜索请求转发到所有的分片(主分片、副本分片)
  * 每个分片在本地执行查询并将结果放到from+size的本地队列中
  * 每个分片返回文档id和文档在队列中的排序值给协调节点，协调节点将这些值合并产生全局排序结果
* 收集(gather阶段)
  * 协调节点根据排序结果中的文档id向相关分片发出get请求取回相关文档
  * 当协调节点取回所有文档后向客户端返回结果

# 映射

## 需求来源

* elasticsearch支持全文搜索，但全文数据在存储前会做分析并提取词项，但文档中的数据并不是都需要如此：比如文档创建时间、文章标题、作者等，这些本来就是结构化数据，不需要进行分析
* 通过预定义字段可以增加数据搜索的维度，提升搜索质量
* 通过预定义字段的数据类型也可以优化存储结构

## es自动判断数据类型

| json type              | field type |
| ---------------------- | ---------- |
| Boolean: true or false | boolean    |
| number: 123            | long       |
| float: 123.45          | double     |
| date: "2014-09-15"     | date       |
| string: "foo bar"      | string     |

## es数据类型

| 主类型  | 子类型                    |
| ------- | ------------------------- |
| string  | text、keyword             |
| number  | byte、shot、integer、long |
| float   | float、double             |
| boolean | boolean                   |
| date    | date                      |

* text：该字段要被全文搜索，可以被分词

* keyword：不能被分词，比如email地址、主机名、状态码和标签；字段可以用于过滤、排序、聚合等操作；keyword字段只能通过精确值搜索到

## mappings操作

* 创建包含mappings的索引

  ```
  PUT /索引
  {
      "settings": {
          "index": {
              "number_of_shards": "2",
              "number_of_replicas": "0"
          }
      },
      "mappings": {
          "person": {
              "properties": {
                  "name": {
                      "type": "text"
                  },
                  "age": {
                      "type": "integer"
                  },
                  "mail": {
                      "type": "keyword"
                  },
                  "hobby": {
                      "type": "text"
                  }
              }
          }
      }
  }
  ```

* 查询索引mappings信息：/索引/_mappings

# 分词

分词：将文本转化为一系列单词的过程，也叫文本分析，在elasticsearch中称作Analysis

例如：我是中国人=》我、是、中国人

分词api：/_analyze【仅用于文本分析测试；text文本存储时会调用分词器(默认standard，可自定义)进行分词】

```
{
    "analyzer": "standard",
    "text": "hello world"
}
{
    "analyzer": "ik_max_word",
    "text": "我是中国人"
}
```

## 中文分词器IK

项目地址：https://github.com/medcl/elasticsearch-analysis-ik

安装：【插件需要在集群的每个节点都安装，安装后重启elasticsearch】

方式1：下载预编译安装包https://github.com/medcl/elasticsearch-analysis-ik/releases

* 创建插件目录：cd your-es-root/plugins/ && mkdir ik
* 解压插件到目录your-es-root/plugins/ik

方式2：使用es插件命令

```
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.5.4/elasticsearch-analysis-ik-6.5.4.zip
```

## 自定义分词器

例如：创建映射的text类型字段时，自定义分词器

```
{
    "settings": {
        "index": {
            "number_of_shards": "2",
            "number_of_replicas": "0"
        }
    },
    "mappings": {
        "person": {
            "properties": {
                "name": {
                    "type": "text"
                },
                "age": {
                    "type": "integer"
                },
                "mail": {
                    "type": "keyword"
                },
                "hobby": {
                    "type": "text",
                    "analyzer": "ik_max_word"
                }
            }
        }
    }
}
```

# [pipeline/ingest node](https://hacpai.com/article/1512990272091)

预处理：对原始数据进行加工后再存入es集群中，比如使用logstash的grok进行数据处理

ingest node：可以对数据进行预处理的节点，默认所有节点都是ingest node

pipeline：在ingest node中定义的数据处理管道，对经过此管道的数据进行预处理

Processors：对数据加工操作的抽象包装，例如：转换数据类型的Processors：convert

## 创建pipeline

```
PUT /_ingest/pipeline/my-pipeline-id 
{
    "description":"describe pipeline",
    "processors":[
        {
            "convert": {
                "field": "age",
                "type": "integer"
            }
        }
    ]
}
```

## 查看pipeline结构

```
GET /_ingest/pipeline/my-pipeline-id
```

## 使用pipeline处理数据

```
POST /haoke/user?pipeline=my-pipeline-id
{
    "name": "hejingqi",
    "age": 32,
    "sex": "男"
}
```

## 搜索插入结果

```
GET /haoke/user/_search?q=hejingqi
```
# 索引操作

* 创建索引：settings中设置分片数、副本数

  ```
  PUT  /索引
  {
      "settings": {
          "index": {
              "number_of_shards": "2",
              "number_of_replicas": "0"
          }
      }
  }
  ```

* 删除索引：

  ```
  DELETE  /索引
  ```

# 文档操作

## 索引/增加
* 指定id

```
PUT 索引/类型/id
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

* 自动产生id

```
POST  索引/类型
{
    "name": "张三",
    "age": 20,
    "sex": "男"
}
```

## 更新

* 全量更新

  ```
  POST  索引/类型/id
  {
      "name": "张三",
      "age": 25,
      "sex": "男"
  }
  ```

* 局部更新：请求体是doc封装的key/value信息

  ```
  POST  索引/类型/id/_update
  {
      "doc": {
          "name": "李四"
      }
  }
  ```

## 删除

```
DELETE  索引/类型/id
```

## 查询

```
GET  索引/类型/id
```

* 数据格式化：GET  索引/类型/id?pretty
* 返回指定字段：GET  索引/类型/id?_source=字段1,字段2。。。
* 返回源数据：GET  索引/类型/id/_source

### 存在性判断

```
HEAD   索引/类型/id
```

根据http返回码_200/404_判断文档是否存在

## 批量操作

* 批量查询：mget

  ```
  GET  索引/类型/_mget
  {
      "ids": ["y6zSIHQB2fO-rT1snG4Q", "yqw2G3QB2fO-rT1sk26f"]
  }
  ```

* 批量索引、插入、修改、删除：bulk

  ```
  POST  索引/类型/_bulk接口
  {"index":{"_index":"itcast","_type":"person"}}
  {"name":"张三","age":23,"mail":"111@qq.com","hobby":"羽毛球、篮球、足球"}
  {"create":{"_index":"haoke","_type":"user","_id":2001}}
  {"name":"小一","age":23,"sex":"男"}
  {"update":{"_index":"haoke","_type":"user","_id":2001}}
  {"doc":{"name":"小五"}}
  {"delete":{"_index":"haoke","_type":"user","_id":2001}}
  ```
  
  * _bulk接口支持的操作：create、update、delete、index
  * 每两行为一组，第一行定义要操作的文档，第二行定义文档数据【delete时没有第二行】
  * 每行末尾都需要有换行符\n【包含最后一行】
  * index操作时，不需要添加id；没有文档时为create操作、有文档时为update操作。

# 搜索方式

基于_search接口的查询，根据参数位置的不同可以分为：

* 基于URI的搜索(GET)：/_search?q=字符串查询语法
* 基于请求体的搜索(POST)

_search接口直接返回最多10条数据

```
http://49.232.17.71:8200/haoke/user/_search
```

## 基于URI

* 全文(全字段)搜索：直接写搜索的单词

  ```
  /_search?q=alex
  ```

* 字段搜索：在搜索单词之前加上字段名和冒号，比如如果知道单词first 肯定出现在 mesg 字段，可以写作  mesg:first

  ```
  /_search?q=age:25
  ```

* 字段--多条件组合：可以使用  NOT  ,  AND  和  OR  来组合检索，注意必须是大写

  ```
  /_search?q=(name:alex)%20AND%20(age:28)
  ```

* 字段--是否存在：`_exists_:user`  表示要求 user 字段存在， `_missing_:user`  表示要求 user 字段不存在

  ```
  /_search?q=_exists_:name
  ```

- 字段-[范围搜索](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-query-string-query.html#_ranges )：对日期、数字、字符串都可以使用范围搜索。【[]包含边界 {}不包含边界】

  ```
  /_search?q=age:%20[22%20TO%2025]
  ```

* 全文-通配符：用  ?  表示单字符， *  表示0或多个字符

  ```
  /_search?q=alex?
  ```

- 全文-模糊搜索：用  ~  表示搜索单词可能有一两个字母写的不对，比如  frist~

  ```
  /_search?q=alex~
  ```

- 全文-[正则表达式](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html#regexp-syntax )

## 基于请求体-词项

> 此处为基于词项的查询，只对一个字段设置查询条件

* term：单个值匹配：数字、日期、布尔值、以及not_analyzed的字符串(未经分析的text类型)

  ```
  {
      "query": {
          "term": {
              "age": 23
          }
      }
  }
  ```

* terms：多个值匹配(数组)

  ```
  {
      "query": {
          "terms": {
              "age": [23, 20, 28]
          }
      }
  }
  ```

* range：范围查询【操作符：lt/小于、lte/小于等于、gt/大于、gte/大于等于】

  ```
  {
      "query": {
          "range": {
              "age": {
                  "gte": 23,
                  "lt": 30 
              }
          }
      }
  }
  ```

* exists：是否包含某字段

  ```
  {
      "query": {
          "exists": {
              "field": "age"
          }
      }
  }
  ```

* 基于id查询

  ```
  {
      "query": {
          "ids": {
              "values": ["FONNOHQBDlTN8HRyeJoX", "F-NNOHQBDlTN8HRyeJoY"]
          }
      }
  }
  ```

* 模式匹配
  * prefix：前缀模糊匹配
  * wildcard：通配符匹配
  * regexp：正则匹配

# 全文搜索

全文搜索vs词项搜索：全文搜索会对查询条件做分析；使用的分析器可以在创建索引时设置analyzer参数或search_analyzer参数，也可以在搜索时，在搜索接口_search设置analyzer参数

全文搜索两个重要方面：

* 相关性(Relvance)：评价查询和其结果间相关性程度，并根据这种相关性排序；这种计算方式可以是TF/IDF方法、地理位置临近、模糊相似等算法
* 分词(Analysis)：将文本转换为有区别的、规范化的token的过程

## 词项匹配-match单值

```
{
    "query": {
        "match": {
            "hobby": "音乐"
        }
    },
    "highlight": {
        "fields": {
            "hobby": {}
        }
    }
}
```

执行的过程如下：

* 检查字段类型：hobby是一个text类型（指定IK分词器），这意味着查询字符串本身也应该被分词
* 分析查询字符串：将查询字符串“音乐”传入IK分词器中，输出单个项(term)：音乐，因为只有一个单词项，所以match查询执行的是单个term查询
* 查询匹配文档：用term查询在倒排索引中查找包含“音乐”的一组文档
* 为每个文档评分并排序：用term查询计算每个文档的相关度评分_score，这个将词频(term frequency)【即“音乐”在相关文档hobby字段中出现的频率】、反向文档频率(inverse document frequency)【即“音乐”在所有文档的hobby字段出现的频率】、字段长度【即字段越短相关度越高】相结合的计算方式

 查询结果如下：

```
{
    "took": 20,
    "timed_out": false,
    "_shards": {
        "total": 2,
        "successful": 2,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 0.7438652,
        "hits": [
            {
                "_index": "itcast",
                "_type": "person",
                "_id": "F-NNOHQBDlTN8HRyeJoY",
                "_score": 0.7438652,
                "_source": {
                    "name": "赵六",
                    "age": 23,
                    "mail": "444@qq.com",
                    "hobby": "听音乐、看电影"
                },
                "highlight": {
                    "hobby": [
                        "听<em>音乐</em>、看电影"
                    ]
                }
            }
        ]
    }
}
```

## 词项匹配-match多值

* 默认：空格分隔的多个词为“逻辑或”

  ```
  {
      "query": {
          "match": {
              "hobby": "羽毛球 篮球"
          }
      },
      "highlight": {
          "fields": {
              "hobby": {}
          }
      }
  }
  ```

* 设置多个词为“逻辑与“关系

  ```
  {
      "query": {
          "match": {
              "hobby": {
                  "query": "羽毛球 篮球",
                  "operator": "and"
              }
          }
      },
      "highlight": {
          "fields": {
              "hobby": {}
          }
      }
  }
  ```

* 设置相似度百分比minimum_should_match，避免多个词使用极端的and或or方式搜索

  ```
  {
      "query": {
          "match": {
              "hobby": {
                  "minimum_should_match": "70%",
                  "query": "足球 篮球"
              }
          }
      },
      "highlight": {
          "fields": {
              "hobby": {}
          }
      }
  }
  ```

  minimum_should_match：相似度越高，包含的文档越少

  > 相似度设置，需要不断测试，以达到预期结果

  * 100%相当于and
  * 50%相当于or

## 多字段/全文搜索

* multi_match：同时匹配多个字段

  ```
  {
      "query": {
          "multi_match": {
              "query": "音乐",
              "fields": ["name", "hobby"] 
          }
      }
  }
  ```

* query_string：和URI中/_search?q=语法类似，定义字段时在默认字段搜索，不定义字段时进行全文搜索

  ```
  {
      "query": {
          "query_string": {
              "query": "音乐",
              "default_field":  "hobby"
          }
      }
  }
  ```

* simple_query_string：对query_string的查询进行简化，包括：忽略查询时的异常、引入更便捷的简化操作符

## 多条件搜索-bool/match

组合多个查询条件，可用的布尔子类型如下：

* must：查询结果必须要包含内容，相当于and；结果会根据**相关性排序**

* filter：查询结果必须要包含内容；但不影响结果排序

* must_not：查询结果不能包含内容，相当于not；不影响结果排序

* should

  * 单独使用时，多个查询条件至少有一个匹配【参数minimum_should_match调节，设置相关性】；结果会根据**相关性排序**【minimum_should_match为2表示，should中的多个词至少要满足2个】

    ```
    {
        "query": {
            "bool": {
                "should": [
                    {
                        "match": {
                            "hobby": "乒乓球"
                        }
                    },
                    {
                        "match": {
                            "hobby": "篮球"
                        }
                    }
                ],
                "minimum_should_match": 2
            }
        },
        "highlight": {
            "fields": {
                "hobby": {}
            }
        }
    }
    ```

  * 和filter、must共同使用时，不影响查询结果，但影响结果**相关性排序**【包含should条件的排序更高】

    minimum_should_match调节排序

    ```
    {
        "query": {
            "bool": {
                "must": {
                    "match": {
                        "sex": "男"
                    }
                },
                "should": {
                    "match": {
                        "name": {
                            "query": "小",
                            "minimum_should_match": "50%"
                        }
                    }
                }
            }
        },
        "highlight": {
            "fields": {
                "sex": {}
            }
        }
    }
    ```

    权重调节排序

    ```
    {
        "query": {
            "bool": {
                "must": {
                    "match": {
                        "hobby": "足球"
                    }
                },
                "should": [
                    { "match": {
                        "hobby": {
                            "query": "乒乓球",
                            "boost": 10
                        }
                    }},
                    { "match": {
                        "hobby": {
                            "query": "篮球",
                            "boost": 2
                        }
                    }}
                ]
            }
        }
    }
    ```

# 聚合查询

> 类似sql查询中的group by

```
{
    "aggs": {
        "age_avg": {
            "avg": {
                "field": "age"
            }
        }
    }
}
```

* aggs：指定聚合查询
* age_avg：聚合名称
* avg：聚合类型

# 结果处理

* 分页：from：跳过前m个结果，size：返回n条数据

  ```
  {
      "from": m,
      "size": n
  }
  ```
  
* 高亮显示

  ```
  "highlight": {
      "fields": {
          "hobby": {}
      }
  }
  ```

# 信息查看-cat

* [通用参数](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/cat.html#common-parameters)

```
v                                     查看详情
help                                  帮助信息
h=column1,column2                     过滤特定表头header
bytes=b                               数字类型格式化【bytes、size、time】
format=json                           文本格式化【text、json、yaml】
s=column1,column2:desc,column3        排序         
```

* 常见用法

```
# 健康信息
/_cat/health?v            
* green：所有主分片和副本分片都可用
* yellow：主分片都可用，副本分片部分可用
* red：主分片部分可用

# 索引信息
/_cat/indices?v

# 节点信息
/_cat/nodes?v
```

# 过滤VS查询

```
{
    "query": {
        "bool": {
            "filter": {
                "term": {
                    "age": 23
                }
            }
        }
    }
}
{
    "query": {
        "match": {
            "hobby": "足球"
        }
    }
}
```

* filter只过滤结果，结果不会排序，但会缓存【利用缓存特性，可以用于精确匹配的搜索，如term查询】
* query查询结果会根据相关性排序(更耗时)，但不会缓存

# 客户端-python

> python客户端[elasticsearch-py](https://elasticsearch-py.readthedocs.org/)为低级客户端，只封装核心操作对象，需要自己构造json格式的DSL进行CRUD、搜索等操作

* 安装：pip install elasticsearch

  ```
  # Elasticsearch 7.x
  elasticsearch>=7.0.0,<8.0.0
  
  # Elasticsearch 6.x
  elasticsearch>=6.0.0,<7.0.0
  
  # Elasticsearch 5.x
  elasticsearch>=5.0.0,<6.0.0
  
  # Elasticsearch 2.x
  elasticsearch>=2.0.0,<3.0.0
  ```

* 使用

  ```
  from elasticsearch import Elasticsearch
  
  node_list = [
      {"host": "192.168.31.221", "port": 9200},
      {"host": "192.168.31.222", "port": 9200},
      {"host": "192.168.31.223", "port": 9200}
  ]
  es = Elasticsearch(hosts=node_list)
  doc = {
      "name": "赵七",
      "age": 33,
      "mail": "424@qq.com",
      "hobby": "听音乐、看电影、跑步"
  }
  # 索引文档
  # res = es.index('haoke',doc_type='person', body=doc)
  # print(res['result'])
  
  # 根据id查询单个文档
  # res = es.get(index='haoke', doc_type='person', id='0mblTHQBBnpVRgexf4YB')
  # print(res['_source'])
  
  # 刷新索引，使刚变更的文档可查
  # es.indices.refresh(index='haoke')
  
  # 搜索文档
  query_json = {
      "query": {
          "term": {
              "age": 23
          }
      }
  }
  res = es.search(index='haoke', doc_type='person', body=query_json)
  for hit in res['hits']['hits']:
      print("%(name)s %(age)s %(hobby)s" % hit['_source'])
  ```

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

​                    = 原始数据 \*（1 + 副本数）\* 1.7

​                    =  原始数据 \* 3.4         

## 分片

* 数据量和索引数
  * 创建多少索引(定期轮转、删除、合并)
  * 每个索引的数据量
  * 每个索引配置的主分片和副本分片
* 分片大小：
  * 单个分片大小保持在10GB - 50GB之间(30GB最优);日志分析或者超大索引场景，建议单个shard大小不要超过100GB。
  * 分片过大：可能使 ES 的故障恢复速度变慢
  * 分片过小：可能导致非常多的分片，因为每个分片使用一些数量的 CPU 和内存，从而导致读写性能、内存不足等问题。
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