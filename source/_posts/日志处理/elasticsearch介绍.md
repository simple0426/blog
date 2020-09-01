---
title: elasticsearch介绍
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
* 副本：主分片的完全复制品，它和主分片内容完全一样，保证了分片的高可用，同时也可以响应读请求
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

## 文档格式

以json格式存储，由元数据(metadata)和数据(_source)组成

元数据包含：

* _index：文档所在索引
* _type：文档类型，每个类型都有自己的映射(mapping)
* _id：文档id
* _version：文档版本

# 索引

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

* 查询索引mappings：/索引/_mappings

# 分词

分词：将文本转化为一系列单词的过程，也叫文本分析，在elasticsearch中称作Analysis

例如：我是中国人=》我、是、中国人

分词api：/_analyze【仅用于文本分析测试；text文本会自动分词，但可以在创建映射时自定义分词器(如中文分词器IK)】

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

安装：

方式1：下载预编译安装包https://github.com/medcl/elasticsearch-analysis-ik/releases

* 创建插件目录：cd your-es-root/plugins/ && mkdir ik
* 解压插件到目录your-es-root/plugins/ik

方式2：使用es插件命令

```
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.3.0/elasticsearch-analysis-ik-6.3.0.zip
```

## 自定义分词器

创建映射的text字段类型时，自定义分词器

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

# 搜索方式

基于_search接口的查询，根据参数位置的不同可以分为：

* 基于URI的请求方式(GET)：/_search?q=字符串查询语法
* 基于请求体的请求方式(POST)

_search接口直接返回最多10条数据

```
http://49.232.17.71:8200/haoke/user/_search
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

> 结构化查询

* term用于精确匹配：数字、日期、布尔值、以及not_analyzed的字符串(未经分析的text类型)

  ```
  {
      "query": {
          "term": {
              "age": 23
          }
      }
  }
  ```

* terms精确匹配一个数组中的多个值

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

* match：既可以进行单字段精确查询(term)，也可以全文搜索(text分词为term)

  ```
  {
      "query": {
          "match": {
              "name": "张三"
          }
      }
  }
  ```

* prefix：前缀模糊匹配

* wildcard：通配符匹配

* regexp：正则匹配

# 全文搜索

全文搜索两个重要方面：

* 相关性(Relvance)：评价查询和其结果间相关性程度，并根据这种相关性排序；这种计算方式可以是TF/IDF方法、地理位置临近、模糊相似等算法
* 分词(Analysis)：将文本转换为有区别的、规范化的token的过程

## 单词搜索

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

## 多词搜索

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

* 避免多个词使用极端的and或or方式搜索，可以设置相似度百分比minimum_should_match

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

# 多条件查询

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

* 分页-URI方式：size：返回指定条目数，from：跳过前n个结果

  ```
  /_search?size=2&from=1
  ```

* 高亮显示-请求体方式

  ```
  "highlight": {
      "fields": {
          "hobby": {}
      }
  }
  ```

# 过滤(filter)和查询(query)

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