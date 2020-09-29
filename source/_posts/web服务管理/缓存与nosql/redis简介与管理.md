---
title: redis简介与管理
tags:
  - 管理
categories:
  - redis
date: 2019-10-25 11:29:57
---

# [简介][redis-intro]
* redis是一个开源的、基于内存的数据结构存储，但也可以根据需要开启持久化存储（rdb、aof）
* 可以被当做数据库、缓存、消息代理使用
* 使用C语言编写，不需要外部依赖，官方默认支持linux系统，也可以使用在windows上运行
    - [microsoft实现的3.0版本](https://github.com/microsoftarchive/redis)
    - [第三方实现的4/5版本](https://github.com/tporadowski/redis)

## 内置功能
* 主从异步复制
* lua脚本
* 数据过期策略：LRU
* 事务
* 数据持久化策略：rdb、aof
* 高可用：Redis Sentinel 
* 自动分区：redis Cluster
* 发布、订阅、取消订阅（消息代理）：Pub/Sub

## 数据类型
* strings：二进制安全的字符串
* hash：是由字段和其值构成的映射，字段和值都是字符串；和ruby、pyhon中的字典类似
* lists：根据插入顺序排序的字符串元素的集合
* sets：由独一无二的、未排序的元素构成的集合
* sorted sets：排序的集合，每个字符串成员都关联一个浮点数值用于排序
* bitmaps：位图，可以处理字符串中的每一位(设置、清除)，找到第一个设置或未设置的位
* hyperloglogs：这是一个概率数据结构，用于估计集合的基数
* streams
* geospatial indexes

## 支持的原子操作
* 字符串追加
* 增加hash中的值
* 向列表中添加元素
* 计算集合的交集、并集、差集
* 获取排序集合中最高排名的成员

## 下载与安装
* 版本重要提示：从5.0版本开始，redis内置关键词slave切换为replica，[相关issue](https://github.com/redis/redis/issues/5335)
* 下载与安装

```
$ wget http://download.redis.io/releases/redis-5.0.5.tar.gz
$ tar xzf redis-5.0.5.tar.gz
$ cd redis-5.0.5
$ make
```

* 安装错误

```
错误：make安装报错 jemalloc/jemalloc.h: No such file or directory。
原因：开始时gcc未安装，安装gcc后存在make编译残留
解决：make distclean  && make
```

# [复制操作](https://redis.io/topics/replication)

## 同异步
* 异步：redis复制是异步的，主实时发送操作的命令到从机
* 部分同步：主从复制出现异常，从机可以通过部分同步功能(repl-backlog-size)与主机重新建立复制关系
    - 可以通过rdb文件(非aof)实现部分同步功能（例如从机重启或升级时，可以SHUTDOWN(save & quit)保存数据）
* 同步：使用WAIT命令可以实现同步复制

## 主从架构
* 主可以有多个从(从机只读)，这样可以分担主的读压力，同时增加数据安全性
* 从也可以有从，即构成级联架构，此时sub-slave从master接收复制流
* 主从结构
    - 为了安全性，主应当开启数据持久化（rdb、aof），从机也可以开启持久化（rdb）
    - 如果主没有开启持久化，而从开启了持久化时：当主机重启，由于内存数据丢失，重启后复制流也会删除从机的数据

## 非阻塞式
* 在master端，复制是非阻塞的
* 在slave端，大部分情况下，复制也是非阻塞的；
    - 在同步初始化阶段，可以配置允许从机向客户端提供旧的数据(slave-serve-stale-data )以避免阻塞
    - 在同步后，依然会有一小段阻塞时间（4.0之前删除数据、加载新的数据都会阻塞；4.0之后会用单独的线程执行删除操作，但是加载数据依然会阻塞）

## 从机过期key处理
- 不依赖于主从间的同步时钟
- 从主上接收处理过期的命令(del)，然后本地执行此命令
- 为了保持数据一致性，对于应当删除而没有删除的key，从机返回错误
- 在lua脚本执行期间，主的数据会被冻结，不会产生过期删除操作

## 复制选项
* 无盘复制：repl-diskless-sync
    - 默认设置下，为了实现复制功能，即使没有开启持久化设置，也会写rdb文件，除非使用无盘复制功能（repl-diskless-sync）
* 只读设置：slave-read-only
    - 如果从机可写，在4.0之前会存在内存泄漏（key过期，客户端不可见，但依然在内存中）
    - 4.0之后的版本，当数据库序列小于63时，不会出现内存泄漏
    - 4.0之后的版本，从机只是本地可写，即不会将本地变更传递给它的从机（sub-slave从master接收复制流）
* 连接设置：
    - slaveof 192.168.1.1 6379
    - masterauth password
* 设置少于指定数量主机或大于某个延迟时停止写操作
    - min-slaves-to-write 3
    - min-slaves-max-lag 10
* nat和端口转发环境下的配置（docker）
    - slave-announce-ip
    - slave-announce-port

## 状态查看
- INFO replication
- ROLE

# 服务端管理
## [配置和优化](https://redis.io/topics/admin)
* 开启系统swap
* 内存调优：
    - 允许过量使用内存：sysctl vm.overcommit_memory=1
    - 关闭影响内存使用和延迟的因素：echo never > /sys/kernel/mm/transparent_hugepage/enabled
* 内存大小设置：
    - maxmemory限制的内存：数据、数据外的开销、碎片开销、rdb保存和aof重写占用的内存（最多会占用常规使用量的2倍）
    - 复制缓冲区（repl-backlog-size）：确保从机可以更容易的重连（支持部分同步）
* 当使用守护进程工具时(如supervisor、systemd)，设置daemonize为no
* 基于安全性，设置listen和requirepass参数
* 有用的工具
    - 延迟诊断：latency doctor【需要设置latency-monitor-threshold】
    - 内存诊断：memory doctor

## 升级或重启
### 主从模式
* 把要操作的实例作为从机与主机同步
* 检测同步日志，确保初始化同步完成
* 使用INFO命令，确保主从有相同数量的key
* 从机开启写权限：CONFIG SET slave-read-only no
* 停止主机写操作：CLIENT PAUSE
* 通过监控命令【redis-cli monitor】确保主不再接收到任何数据操作命令，此时：
    - 从提升为主：SLAVEOF NO ONE 
    - 旧主机关闭

### 高可用或集群模式
* 常规流程：升级从机，执行主从手动切换
* 注意：由于4.0版本在集群总线协议级别与3.2版本不兼容，此时集群需要整体重启；5.0版本集群总线向后兼容4.0版本

# [客户端管理](https://redis.io/topics/clients)
## 参数设置
* 最大连接数：maxclients；与os最大文件描述符相关，同时需要保留部分描述符用于redis内部使用
    - redis可以设置最大文件描述符时，所需的最大文件描述符为：fd=maxclients+32(内部使用)
    - redis不可设置最大文件描述符时(即maxclients大于最大文件描述符)，maxclients=fd-32
* 输出缓冲区限制：
    - client-output-buffer-limit normal 0 0 0
    - client-output-buffer-limit slave 256mb 64mb 60
    - client-output-buffer-limit pubsub 32mb 8mb 60
* 查询缓冲区：client-query-buffer-limit，主要用于避免客户端bug引起服务端崩溃；默认1GB，一般不做限制。
* 客户端超时：timeout，不是很精确的计算，只用于一般类型的客户端（非repl、pub/sub类型）
* tcp心跳设置：tcp-keepalive，避免客户端"假死"(虽然连接，但是已不可用)

## 管理命令
* client list：显示客户端列表，参数含义如下：
    - addr：客户端ip和port
    - name：客户端名称
        + client setname：设置当前会话名称
        + client getname：获取当前会话名称
    - age：连接存在时间
    - idle：连接空闲时间
    - flags：连接类型
        + S：复制客户端
        + N：一般客户端
    - omem：已使用的输出缓冲区大小
    - cmd：最近执行的命令
* client kill：关闭客户端连接

[redis-intro]:https://redis.io/topics/introduction
