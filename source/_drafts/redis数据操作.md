---
title: redis数据操作
tags:
categories:
---
# [数据类型及抽象](https://redis.io/topics/data-types-intro)
## redis键
* 不建议使用非常长的key，因为在查找比较时需要消耗更多的资源；如果确实需要存储较长的key，可以存储key的hash值
* 也不建议使用较短的key，较短的key可读写太差
* key定义时的格式要统一，比如使用冒号、点、横杠等
* key的最大长度512MB

## redis字符串
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
* ltrim key start end：只保留start到end的值
* lpop key：弹出列表左侧的值
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
* sinter key key1：查看集合间的交集
* srandmember key：获取集合中随机一个成员

## 有序集合
### 特点
* 与集合对比，有序集合的每个成员都关联一个分数值，这个分数值用于排序
* 可以通过分数、或索引获取一个范围内的成员

### 命令
* zadd key socre member：向有序集合key中添加分数为socre的成员member
* zrange key start end [withscores]：获取有序集合的成员
* zrank key member：获取成员的排序(索引)
* zrangebyscore key min max：获取socre在min和max之间的成员

## 哈希/字典
* hmset key field value [field value ...]：一次设置字典的多个字段
* hset key field value：一次设置字典的一个字段
* hgetall key：一次返回字典的所有字段
* hget key field：一次返回字典中某个字段的值

## 位图
* setbit key offset value：设置offset处为0或1
* getbit key offset：获取offset处的值

## HyperLogLog
* 应用参考：https://www.cnblogs.com/ysuzhaixuefei/p/4052110.html

# 管理命令
* exists key：判断键是否存在
* type key：判断键的数据类型
* del key：删除特定key
* keys：返回特定匹配的key列表
* flushdb：删除当前库的所有key
* flushall：删除所有数据库的key

# 暂存：https://redis.io/topics/data-types-intro#redis-lists