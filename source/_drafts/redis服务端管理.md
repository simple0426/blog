---
title: redis服务端管理
tags:
categories:
---
# [复制](https://redis.io/topics/replication)
## 同异步
* 异步：redis复制是异步的，主实时发送操作的命令到从机
* 部分同步：主从复制出现异常，从机可以通过部分同步功能repl-backlog-size与主机重新建立复制关系
    - 可以通过rdb文件（非aof）实现部分同步功能（例如从机重启或升级时，可以SHUTDOWN（save & quit）保存数据）
* 同步：使用WAIT可以支持同步复制

## 主从架构
* 主可以有多个从（从机只读），这样可以分担主的读压力，同时增加数据安全性
* 从也可以有从，即构成级联架构，此时sub-slave从master接收复制流
* 主从结构
    - 为了安全性，主应当开启数据持久化（rdb、aof），从机也可以开启持久化（rdb）
    - 如果主没有开启持久化，而从开启了持久化时：当主机重启，由于内存数据丢失，重启后复制流也会删除从机的数据

## 非阻塞式
* 在master端，复制是非阻塞式的
* 在slave端，大部分情况下，复制也是非阻塞的；
    - 在同步初始化阶段，可以配置允许从机向客户端提供旧的数据（slave-serve-stale-data ）以避免阻塞
    - 在同步后，依然会有一小段阻塞时间（4.0之前删除数据、加载新的数据都会阻塞；4.0之后会用单独的线程执行删除操作，但是加载数据依然会阻塞）


## 从机过期key处理
- 不依赖于主从间的同步时钟
- 从主上接收处理过期的命令（del）
- 为了保持数据一致性，对于应当删除而没有删除的key，从机返回错误
- 在lua脚本执行期间，主的数据会被冻结，不会产生过期删除操作

## 复制选项
* 无盘复制：repl-diskless-sync
    - 默认设置下，为了实现复制功能，即使没有开启持久化设置，也会写rdb文件；除非使用无盘复制功能（repl-diskless-sync）
* 只读设置：slave-read-only
    - 如果从机可写，在4.0之前会存在内存泄漏（key过期，客户端不可见，但依然在内存中）
    - 4.0之后的版本，当数据库序列小于63时，不会出现内存泄漏
    - 4.0之后的版本，从机只是本地可写，即不会将本地变更传递给它的从机（sub-slave从master接收复制流）
* 连接设置：
    - slaveof 192.168.1.1 6379
    - masterauth password
* 少于指定数量主机或大于某个延迟时停止写操作
    - min-slaves-to-write 3
    - min-slaves-max-lag 10
* nat和端口转发环境下的配置（docker）
    - slave-announce-ip
    - slave-announce-port

## 状态查看
- INFO replication
- ROLE

# [配置和优化](https://redis.io/topics/admin)
* 设置系统使用swap
* 内存调优：
    - 设置允许过量使用内存：sysctl vm.overcommit_memory=1
    - 关闭影响内存使用和延迟的因素：echo never > /sys/kernel/mm/transparent_hugepage/enabled
* 内存大小设置：
    - maxmemory限制的内存：数据、数据外的开销、碎片开销、rdb保存和aof重写占用的内存（最多会占用常规使用量的2倍）
    - 复制缓冲区（repl-backlog-size）：确保从机可以更容易的重连（支持部分同步）
* 当使用守护进程工具时（如supervisor、systemd），设置daemonize为no
* 基于安全性，应当设置listen和requirepass参数
* 有用的工具
    - 延迟诊断：latency doctor【latency-monitor-threshold】
    - 内存诊断：memory doctor

# 服务端升级或重启
## 主从模式
* 把要操作的实例作为从机与主机同步
* 检测同步日志，确保初始化同步完成
* 使用INFO命令，确保主从有相同数量的key
* 从机开启写权限：CONFIG SET slave-read-only no
* 停止主机写操作：CLIENT PAUSE
* 通过监控命令【redis-cli monitor】确保主不再接收到任何数据操作命令，此时：
    - 从提升为主：SLAVEOF NO ONE 
    - 旧主机关闭

## 高可用或集群模式
* 常规流程：升级从机，执行主从手动切换
* 注意：由于4.0版本的在集群总线协议级别与3.2版本不兼容，此时集群需要整体重启；5.0版本集群总线向后兼容4.0版本
