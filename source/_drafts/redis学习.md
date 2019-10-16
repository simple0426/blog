---
title: redis学习
tags:
categories:
---
# [简介][redis-intro]
* redis是一个开源的、基于内存的数据结构存储，但也可以根据需要开启持久化存储（rdb、aof）
* 可以被当做数据库、缓存、消息代理使用；
* 使用C语言编写，不需要外部依赖，官方默认支持linux系统，也可以使用[microsoft实现的版本][windows-redis]在windows上运行

# 内置功能
* 主从异步复制
* lua脚本
* 数据过期策略：LRU
* 事务
* 数据持久化策略：rdb、aof
* 高可用-Redis Sentinel 
* 自动分区-redis Cluster
* 发布、订阅、取消订阅：Pub/Sub【消息代理】

# 数据类型
* strings：字符串
* hash：哈希【字典】
* lists：列表
* sets：集合
* sorted sets：带范围查询的排序集合
* bitmaps
* hyperloglogs
* streams
* geospatial indexes

# 支持的原子操作
* 字符串追加
* 增加hash中的值
* 向列表中添加元素
* 计算集合的交集、并集、差集
* 获取排序集合中最高排名的成员

# 下载与安装
* 下载：http://download.redis.io/releases/redis-4.0.14.tar.gz
* 安装

```
$ wget http://download.redis.io/releases/redis-5.0.5.tar.gz
$ tar xzf redis-5.0.5.tar.gz
$ cd redis-5.0.5
$ make
```

# [redis-cli][rediscli]
## 非交互模式
* 命令行使用：redis-cli incr mycounter
    - 重定向输出：redis-cli incr mycounter > /tmp/output.txt
    - 额外信息控制：
        + --raw：只显示命令结果
        + --no-raw：显示操作的数据类型及操作结果【这是默认模式，但是客户端也会判断如果有tty终端则显示数据类型，没有tty（重定向）则不显示数据类型】
* 连接
    - -h：连接主机
    - -p：连接端口，默认6379
    - -a：连接密码
    - -n：指定指定连接的库【默认连接数据库0】
    - -u：使用redis协议通过URI连接，如：redis-cli -u redis://password@redis-16379.hosted.com:16379/0 ping
* 重定向输入：
    - -x：读取最后一个参数：redis-cli -x set foo < /etc/services
    - 执行文件内的命令：cat /tmp/commands.txt | redis-cli
    ```
    set foo 100
    incr foo
    append foo xxx
    get foo
    ```
* 重复执行命令
    - -r：执行次数
    - -i：时间间隔，单位是秒，最小为0.1，默认为0
* 批量插入：--pipe【cat data.txt | redis-cli --pipe】
* 导出csv格式：--csv【redis-cli --csv lrange mylist 0 -1】

## 交互模式
>默认可以浏览历史命令、命令补全功能

- 选择数据库：select 2
- 数据库大小：dbsize
- 认证：auth
- 连接其他实例：connect
- 执行命令n次：n command
- 帮助信息：
    + 分类帮助：`help @<category>`
    + 特定命令帮助：help command
- 清屏：clear

## 特殊模式
* 状态监控
    - 参数：--stat （-i interval）
    - 范例：redis-cli --stat
* 数据库中扫描出最大key
    - 参数：--bigkeys
    - 范例：redis-cli --bigkeys
* 获取key列表
    - 参数：--scan【对比交互模式下`keys *`，scan参数以非阻塞方式获取key列表】
    - 范例：redis-cli --scan | head -10
    - 模式匹配参数：--pattern
    - 模式匹配范例：`redis-cli --scan --pattern '*-11*'`
* 发布订阅模式
    - 订阅：redis-cli psubscribe '*'
    - 发布：redis-cli PUBLISH mychannel mymessage
* 监控实例执行的命令：redis-cli monitor
* 用不同方式检测服务端实例延迟
    - --latency：每秒100次向服务端发送ping信号
    - --latency-history：每15秒开启一个新的会话连接用于压测【-i指定时间间隔】
    - --latency-dist：以带颜色的光谱图形式显示延迟信息
* 检测客户端延迟： --intrinsic-latency 【test-time】：redis-cli客户端所在主机的固有延迟【如内核调度，虚拟化开销等】
* 远程备份rdb
    - 参数：--rdb
    - 范例：redis-cli -h 172.15.22.9 -a 123456 --rdb /tmp/dump.rdb
* 测试复制模式下从机接收的内容
    - 参数：--slave
    - 范例：redis-cli -h 172.15.22.9 -a 123456 --slave
* 测试不同LRU策略下的key命中率
    - 影响因素：测试样本数量、最大可用内存(maxmemory )
    - 参数：--lru-test sample-number
    - 范例：./redis-cli --lru-test 10000000

[windows-redis]: https://github.com/microsoftarchive/redis
[rediscli]: https://redis.io/topics/rediscli
[redis-intro]:https://redis.io/topics/introduction