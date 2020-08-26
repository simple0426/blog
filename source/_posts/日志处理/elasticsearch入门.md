---
title: elasticsearch入门
tags:
  - elasticsearch
  - search
categories:
  - elk
date: 2020-08-25 11:09:25
---


# 基本概念

* 集群：多个协同工作的节点构成的集合
* 节点：运行es软件的java进程
* 索引(index)：由分片构成的逻辑集合，也是文档存储在物理上的区分，相当于mysql的库
* 分片(shard)：将索引的数据分解为大小合适的片段(分片)分散在不同节点进行存储，这样可以提升数据读写速度、扩充集群存储容量
* 副本：主分片的完全复制品，它和主分片内容完全一样，保证了分片的高可用
* 文档(doc)：一条记录，是es中数据存储和检索的基本单位；相当于mysql的一行记录
* 映射(mapping)：将文档转变为索引的过程
  * 将文档中的关键字(term)提取出来，即文档分析(analysis)
  * 将关键字与文档的对应关系保存起来
  * 对关键字本身做索引排序
* 字段(field：一个文档由若干个字段组成，每个字段可以有不同的数据类型
  * 基本类型：比如字符串、数值、日期、布尔、字节、范围
  * 扩展类型：数组、字典(对象)
  * 多数据类型：让一个字段同时具备多种数据类型特征
* 分词/项(term)：索引的最小单位；
  * 某个field对应的内容如果是全文检索类型，会将内容进行分词为若干term
  * 如果是不分词的字段，那么该字段的内容就是一个term

## segment(分段)

* elasticsearch中的每个分片包含多个segment，每一个segment都是一个倒排索引；在查询的时，会把所有的segment查询结果汇总归并后最为最终的分片查询结果返回;    
* 在创建索引的时候，elasticsearch会把文档信息写到内存中（为了安全，也一起写到translog），定时（可配置）把数据写到segment缓存小文件中，然后刷新查询，使刚写入的segment可查.
* 虽然写入的segment可查询，但是还没有持久化到磁盘上。因此，还是会存在丢失的可能性的. 所以，elasticsearch会执行flush操作，把segment持久化到磁盘上并清除translog的数据
* 当索引数据不断增长时，对应的segment也会不断的增多，查询性能可能就会下降。因此，Elasticsearch会触发segment合并的线程，把很多小的segment合并成更大的segment，然后删除小的segment。   
* segment是不可变的，当我们更新一个文档时，会把老的数据打上已删除的标记，然后写一条新的文档。在执行flush操作的时候，才会把已删除的记录物理删除掉。

## 倒排索引(inverted index)

>  对于关系型数据库，一般是针对主键建立索引，然后索引指向数据内容

倒排索引是针对文档内容建立索引，然后索引指向文档，实现了term到doc list的映射

## 文档

以json格式存储，由元数据(metadata)和数据(_source)组成

元数据包含：

* _index：文档所在索引
* _type：文档类型，每个类型都有自己的映射(mapping)
* _id：文档id
* _version：文档版本

# 索引管理

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
  
* 删除索引：DELETE  索引


# 数据操作

## 增加
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
DELETE  索引/类型/id

## 查询
GET 索引/类型/id

## 自定义查询

* 数据格式化：GET  索引/类型/id?pretty
* 返回指定字段：GET  索引/类型/id?_source=字段1,字段2。。。
* 返回源数据：GET  索引/类型/id/_source

* 判断文档是否存在：HEAD 索引/类型/id【根据http返回码200/404判断文档是否存在】

## 批量操作

* 批量查询

  ```
  GET  索引/类型/_mget
  {
      "ids": ["y6zSIHQB2fO-rT1snG4Q", "yqw2G3QB2fO-rT1sk26f"]
  }
  ```

* 批量插入、修改、删除

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

  * 每两行为一组，第一行定义要操作的文档，第二行定义文档数据【delete时没有第二行】
  * 每行末尾都需要换行符\n【包含最后一行】
  * 操作类型：create、update、delete、index
  * index操作时，不需要添加id；没有文档时为create操作；有文档时为update操作。

# 搜索

基于_search接口的查询，根据请求方式(GET/POST)的不同可以分为：

* 基于URI的请求方式(GET)：_search?q=字符串查询语法
* 基于请求体的请求方式(POST)

_search接口直接返回最多10条数据

```
http://49.232.17.71:8200/haoke/user/_search
```

## 功能

* 分页：size：返回指定条目数，from：跳过前n个结果

```
http://49.232.17.71:8200/haoke/user/_search?size=2&from=1
```

## URI搜索方式

* 全字段搜索：直接写搜索的单词

    ```
    /_search?q=alex
    ```

* 单字段检索：在搜索单词之前加上字段名和冒号，比如如果知道单词first 肯定出现在 mesg 字段，可以写作  mesg:first

    ```
    /_search?q=age:25
    ```

* 多个条件组合：可以使用  NOT  ,  AND  和  OR  来组合检索，注意必须是大写

    ```
    /_search?q=(name:alex)%20AND%20(age:28)
    ```

* 字段是否存在：`_exists_:user`  表示要求 user 字段存在， `_missing_:user`  表示要求 user 字段不存在

    ```
    /_search?q=_exists_:name
    ```

* 通配符：用  ?  表示单字符， *  表示0或多个字符

    ```
    /_search?q=alex?
    ```

- 模糊搜索：用  ~  表示搜索单词可能有一两个字母写的不对，比如  frist~

  ```
  /_search?q=alex~
  ```

- [范围搜索](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-query-string-query.html#_ranges )：对日期、数字、字符串都可以使用范围搜索。【[]包含边界 {}不包含边界】

  ```
  /_search?q=age:%20[22%20TO%2025]
  ```

- [正则表达式](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html#regexp-syntax )

## 请求体搜索方式

* 全文搜索-match

  ```
  POST /_search
  {
      "query": {
          "match": {
              "age": 22
          }
      }
  }
  ```

* 多条件组合查询

  ```
  POST /_search
  {
      "query": {
          "bool": {
              "must": {
                  "match": {
                      "sex": "男"
                  }
              },
              "filter": {
                  "range": {
                      "age": {
                          "gt": 24
                      }
                  }
              }
          }
      }
  }
  ```
  
  bool：混合多个查询条件
  * must：查询结果必须包含的内容
  * filter：查询结果必须包含的内容，影响结果排序
  
* range：范围查询

* exists：字段是否存在

* prefix：前缀模糊匹配

* wildcard：通配符匹配

* regexp：正则匹配

## 聚合查询

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

# 映射

elasticsearch可以自动判断字段及其数据类型，但是有时候自动判断的类型和实际需求不符合(比如，搜索的字段内容是否需要分词)，这个时候就需要在创建索引的时候指定字段的数据类型(映射)

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

*  查询索引mappings：/索引/_mappings


