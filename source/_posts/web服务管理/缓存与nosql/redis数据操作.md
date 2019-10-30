---
title: redis数据操作
tags:
  - 数据类型
  - 操作命令
categories:
  - redis
date: 2019-10-30 17:34:53
---

# [数据类型及抽象](https://redis.io/topics/data-types-intro)
## key定义
* 不建议使用非常长的key，因为在查找比较时需要消耗更多的资源；如果确实需要存储较长的key，可以存储key的hash值
* 也不建议使用较短的key，较短的key可读写太差
* key定义时的格式要统一，比如使用冒号、点、横杠等
* key的最大长度512MB

## 字符串
* 字符串是redis key最简单的数据类型
* 字符串类型在抓取html碎片和页面时非常有用
* 字符串中可以包含各种数据（包含二进制数据），所以可以在值中存储jpeg等的图片内容
* 字符串长度最大值512MB

## key过期设置
* 支持秒和毫秒级别设置
* 过期的设置以日期形式保存在磁盘或被复制
* 服务停止，时间也会被计算
* 独立操作命令
    * expire key seconds：设置key过期
    * pexpire key milliseconds：设置key过期
    * persist key：取消过期设置
    * ttl：查看过期时间（秒）
    * pttl：查看过期时间（毫秒）

## 列表
* redis列表通过链接列表实现
* 列表可以实现快速的插入数据，而有序集合(sorted sets)则可以实现快速的访问集合中间位置的元素
* 使用ltrim可以始终只保留一定数量的元素：ltrim mylist 0 999
* 列表的阻塞操作（blpop/brpop）可以实现队列功能
    - 阻塞命令(brpop/blpop)一次可以操作多个列表、阻塞时间可以设置为0(永不超时)
    - 阻塞命令返回一个两元素的数组（列表名、元素）
* 一般用法
    - 在社交网络展示最新发布的文章
        + 使用lpush将最新文章的id放入列表
        + 使用lrange 0 9展示最新发布的10篇文章
    - 进程间的通信（生产者-消费者模型）

# [数据类型](https://redis.io/topics/data-types)
## 字符串
* set key value：设置key
    - ex seconds：设置键的过期秒数
    - px milliseconds：设置键的过期毫秒数
    - nx：键不存在才能对键操作
    - xx：键存在才能对键操作
* mset key value key1 value1：一次设置多个key
* mget key key1：一次获取多个值
* get key：获取key的值
* getset key：设置key并返回key的旧值
* incr counter：计数器加1
* decr counter：计数器减1
* incrby counter n：计数器加n
* decrby counter n：计数器减n
* getrange key start end：获取子串
* setrange key index value：从索引处开始替换子串

## 列表
* lpush/rpush key value：从列表左或右侧向列表添加值
* lrange key start end：获取列表从start到end的值
    - 索引可以是负值，比如-1表示最后一个元素
    - 索引从0开始，0表示第一个元素
* llen key：返回列表长度
* ltrim key start end：只保留start到end的值
* lpop/rpop key：弹出列表左侧或右侧的元素
* blpop key key1... timeout：阻塞式弹出列表的值

## 集合
### 特点
* 成员间无固定顺序
* 成员不重复
* 支持交集、并集、差集
* 成员最大数量2^32-1（512MB）

### 命令
* sadd key value：集合添加成员
* smembers key：查看集合成员
* sismember key member：查看元素是否是集合成员
* scard key：获取集合成员数量
* sinter key key1：查看集合间的交集
* sunionstore des_key key key1：获取集合间的并集并存储在新的key中
* spop key：弹出集合中的成员
* srandmember key：获取集合中随机一个成员

## 有序集合
### 特点
* 与集合对比，有序集合的每个成员都关联一个分数值，这个分数值用于排序
* 可以通过分数、或索引获取一个范围内的成员

### 命令
>特殊定义：【-inf：负无穷】【inf：正无穷】

* zadd key socre member：向有序集合key中添加分数为socre的成员member
* zrank key member：获取成员索引位置
    - zrevrank key member：成员的倒序
* zrange key start end [withscores]：根据索引区间获取成员
    - zrevrange key start end：反向排序
* zrangebyscore key min max：根据socre区间获取成员
    - zremrangebyscore key start end：根据score删除成员
* zrangebylex key min mex：根据key的首字符区间获取成员
    - `(`表示开区间、`[`表示闭区间
    - 范例：ZRANGEBYLEX hackers [B [P【key首字母在B和P之间的key】

## 哈希/字典
* hmset key field value [field value ...]：一次设置字典的多个字段
* hset key field value：一次设置字典的一个字段
* hgetall key：一次返回字典的所有字段及其值
* hget key field：一次返回字典中某个字段的值
* hmget key field1 field2：一次返回字典中多个字段的值
* hincrby key field n：对字典中的某个字段自增n

## 位图
* setbit key offset value：设置offset处为0或1
* getbit key offset：获取offset处的值
* bitop operation destkey key：在字符串之间进行位操作（AND, OR, XOR and NOT.）
* bicount key：统计键中设置为1的位的数量
* bitpos key bit start end：返回键中1或0第一次出现的索引位置

## HyperLogLog
* 用途：主要用于唯一性事务的概率统计
* 语法：
    - pfadd key element element1：添加元素
    - pfcount key key1:唯一性统计
* 应用范例：https://www.cnblogs.com/ysuzhaixuefei/p/4052110.html

# 管理命令
* type key：判断键的数据类型
* exists key：判断键是否存在
* del key：删除特定key
* keys foo*：返回特定匹配的key列表
* flushdb：删除当前库的所有key
* flushall：删除所有数据库的key

-----
# [redis命令参考](http://redisdoc.com/index.html)
