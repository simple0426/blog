---
title: redis主从高可用-sentinel
date: 2020-09-29 23:52:37
tags: ['sentinel']
categories: ['redis']
---

# [sentinel功能](https://redis.io/topics/sentinel)

* 监控：持续监控master和slave是否处于工作状态
* 通知：通过API通知系统管理员或程序某个redis实例故障
* 自动故障转移：当master故障时，自动将其中的一个slave提升为master，并配置其他slave连接新的master；当有客户端连接时，通知通知客户端使用新的地址
* 配置提供者：向客户端提供服务发现功能：即master的地址

# 快速部署
## 运行sentinel

```
redis-sentinel /path/to/sentinel.conf
或：redis-server /path/to/sentinel.conf --sentinel
```

配置文件是必须的，因为在重启时要保存集群状态；没有配置文件或配置文件不可写，sentinel将拒绝启动
sentinel默认监听在26379端口，用于和其他sentinel实例以及客户端通信

## 部署前的基础知识

1. 需要至少3个实例保证部署的健壮性，并需要保证它们不在一个故障区
2. Sentinel + Redis的分布式系统不保证在故障转移期间已确认的写操作数据也保留，因为主从是异步复制的；但是一些部署方式可以限制写操作丢失只在某些特定的时刻，当然也会有一些其他不安全的部署方式
3. 需要客户端支持sentinel功能；大部分都支持，并非全部
4. HA设置需要不断测试才能保证安全，因为错误的配置可能只在某些特定的情况下才会发挥作用
5. sentinel、docker和其他形式的nat、端口转换被混合使用时都应该注意：docker默认的端口映射行为，破坏了sentinel对其他sentinel进程的自动服务发现、以及获取redis master的slave列表；应该查询sentinel和docker的相关内容进行设置

## 配置sentinel

sentinel.conf 是一个自文档配置文件，典型的最小配置如下

```
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 60000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1

sentinel monitor resque 192.168.1.3 6380 4
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 5
```

只需要设置要监控的master节点，并为每个独立的master节点设置不同的名称；不需要设置slave节点，这会通过自动发现实现；sentinel会用额外的信息自动更新关于slave的配置，为了在重启时依然保留相关配置信息。
每次在故障转移期间将slave提升为master时，以及每次发现新的Sentinel时，都会重写配置；以上配置监控了两组redis实例，每个实例都包含了数量不定的slave；一组实例叫mymaster，另一组叫resque。

sentinel monitor语句参数的含义如下

```
sentinel monitor <master-group-name> <ip> <port> <quorum>
```

第一行告诉sentinel监控一个叫mymaster的master，地址127.0.0.1，端口6379，选举需要的最小成员数2；
quorum参数含义如下：

* 标记master不可用所需sentinel的最小数量
* quorum只用于检测master故障；为了执行故障转移，需要将一个sentinel选举为故障转移负责人，然后才有权继续进行故障转移操作，而这种选举只发生在具有多数sentinel进程可以互相通信的情况下。

对于有5个sentinel进程的例子，quorum设置为2表示如下含义：

* 如果有2个sentinel同意master不可达，则其中的2个将尝试启动故障转移
* 如果有至少3个sentinel同意mster不可达，并且可以互相通信，则故障转移被授权并实际执行

实际上，这意味着在发生故障期间，如果大多数Sentinel进程无法进行对话（在少数分区中也没有故障转移），则Sentinel不会启动故障转移。

## 其他sentinel选项

其他sentinel选项的格式一般如下

```
sentinel <option_name> <master_name> <option_value>
```

一些选项的作用如下：

* down-after-milliseconds：sentinel认为实例不可达(不回复ping或回复错误)需要的时间，以毫秒计
* parallel-syncs：设置故障转移期间同时向新master进行数据同步的slave数量；这个slave数量越少，则故障转移完成需要的时间也就越多（一次同步一个，直到所有副本节点同步完成）；尽管多数情况下复制是非阻塞的，但也有一些情况会造成副本节点不可用；当设置为1时，可以保证复制期间仅有1个副本节点不可用，而此时未完成同步的其他副本节点可以使用旧数据向客户端提供服务。

可以使用SENTINEL SET命令实时修改参数

## 避免写丢失的配置

> redis.conf

```
min-replicas-to-write 1
min-replicas-max-lag 10
```

当master可以写入至少1个slave中，且收到异步确认的消息不超过10s时，master才可以继续接受客户端的写操作；这样可以防止在发生故障转移时，旧的master依然接受写操作造成部分写操作丢失。

## sentinel/docker/NAT问题

端口和地址映射对sentinel的影响：

* sentinel通过自动发现和其他sentinel协同工作，由于端口和地址映射，sentinel声明的地址和端口对其他sentinel而言是不正确且不能连接的
* sentinel通过在master使用命令INFO检测slave的连接信息，同样由于端口和地址映射，sentinel获取的slave连接信息是不正确的且不可达，最终会因为没有可用的slave无法完成故障切换工作

对于第一个问题可以使用如下配置，使sentinel声明一个可以连接的地址和端口信息

```
sentinel announce-ip <ip>
sentinel announce-port <port>
```

对于第二个问题可以使用如下配置，强制slave向master声明一个可以连接的地址和端口【v3.2.2之后】

```
replica-announce-ip 5.5.5.5
replica-announce-port 1234
```

相应地，集群模式下也有相关的配置

```
cluster-announce-ip 10.1.1.5
cluster-announce-port 6379
cluster-announce-bus-port 6380
```

当然为了操作简单，也可以直接使用docker的host网络模式，直接使用宿主机的网络。

# 集群状态检查

在本机运行3个sentinel实例(端口 5000, 5001, 5002)，同时运行一个master(6379)和一个slave(6380)，使用127.0.0.1进行相互连接
sentinel配置如下

```
port 5000
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

* 从sentinel处检查redis master运行状态

```
# redis-cli -p 5000
127.0.0.1:5000> sentinel master mymaster
```

* 检查其他sentinel和redis slave的信息

```
SENTINEL replicas mymaster
SENTINEL sentinels mymaster
```

* 查看redis master的地址

```
SENTINEL get-master-addr-by-name mymaster
```

#  配置修改和查看

API功能：查看自身的状态、检查被监控的redis master和slave的状态、订阅特定的事件、实时变更配置

## 查询sentinel状态

```
PING # 返回PONG信息
SENTINEL masters                      # 显示被监控的多个master列表
SENTINEL master <master name>         # 显示指定master信息
SENTINEL replicas <master name>       # 显示指定master的slave列表和信息
SENTINEL sentinels <master name>      # 显示监控指定master的sentinel列表和信息
SENTINEL get-master-addr-by-name <master name> # 显示指定master的ip和端口
SENTINEL reset <pattern>              # 重置匹配(glob匹配)的master的所有信息（包含slave和sentinel信息）
SENTINEL failover <master name>       # 强制进行故障转移，无需其他sentinel同意
SENTINEL ckquorum <master name>       # 检测当前sentinel的配置是否满足故障转移的条件
SENTINEL flushconfig                  # 强制将当前sentinel的状态写到磁盘
```

## 实时修改sentinel配置

> 需要在多个sentinel上修改

```
SENTINEL MONITOR <name> <ip> <port> <quorum> # 监控一个新的master主从集群，ip选项不能时主机名
SENTINEL REMOVE <name>                       # 移除一个master主从集群
SENTINEL SET <name> <option> <value>         # 指定master的参数配置
```

## 添加或移除sentinel

**添加**：通过sentinel的自动发现功能，可以直接添加sentinel监控活动的master；为了保证在添加过程中，即使出现网络分区时也可以进行故障转移，应该一个一个的添加新的sentinel，间隔30s以上

**删除**：由于sentinel不会主动删除发现的sentinel信息，即使sentinel不可达；为了保证故障转移的可用，同时又不想动态修改故障转移的条件(quorum)，我们应当在没有网络分区的情况下执行下列步骤：

* 停止任意的sentinel
* 在所有可用sentinel实例执行`SENTINEL RESET *`命令【此命令是重置监控所有master的sentinel信息，也可以指定master】
* 在所有可用sentinel实例执行`SENTINEL MASTER mastername`，以使显示的sentinel信息和活动的sentinel保持一致

## 移除旧的master或slave

sentinel不会主动删除保存的master下的slave信息，为了删除不可用的slave或旧的master，应该在所有sentinel上执行`SENTINEL RESET mastername`，这样可以使sentinel通过INFO命令重新获取master下的slave信息。

## slave优先级

slave的设置_replica-priority_用于决定多个slave在故障转移中的优先级：

* replica-priority设置为0，slave永远不会提升为master
* replica-priority数值越小优先级越高

## redis和sentinel认证

redis认证：在master和所有slave中设置指令以启用认证功能

* requirepass _string_：启用客户端连接的授权密码
* masterauth _string_：在slave中设置连接master的认证密码

同时在master和slave中都设置这两个参数，以保障master与外部的任何连接(客户端、slave、sentinel)都需要认证、slave与外部的任何连接(客户端、sentinel)也需要认证、故障转移后master和slave角色转换后依然保持认证功能

如果需要一个可以匿名访问的slave，可以设置如下：

* 只设置masterauth，不设置requirepass
* 设置replica-priority为0，此slave永远不能提升为master

在redis启用认证的情况下，应该设置sentinel的认证参数：

```
sentinel auth-pass <master-group-name> <pass>
```

## sentinel实例认证

从5.0.1之后，也可以开启sentinel自身的认证：`requirepass "your_password_here"`，
这个认证同时用于sentinel和其他sentinel实例，以及sentinel和客户端的认证
为了使sentinel运行正常，需要在所有sentinel实例配置此参数
这个参数也需要客户端库的支持。

# 精简配置范例

3个redis(1master 2slave)、3个sentinel

* redis最简配置

  ```
  bind 0.0.0.0
  port 6379
  daemonize yes
  dbfilename "dump.rdb"
  dir "./"
  requirepass "123456"
  masterauth "123456"
  min-replicas-to-write 1
  min-replicas-max-lag 10
  # replicaof 192.168.31.232 6379
  # replica-priority 50
  ```

* sentinel最简配置

  ```
  port 5000
  daemonize yes
  dir "/tmp"
  sentinel monitor mymaster 192.168.31.232 6379 2
  sentinel auth-pass mymaster 123456
  ```

# python客户端使用

```
# pip install redis
from redis.sentinel import Sentinel
sentinel = Sentinel([
    ('192.168.31.231', 5000),
    ('192.168.31.232', 5000),
    ('192.168.31.233', 5000)],
    socket_timeout=0.5)

# 获取master的连接字符串（主要用于观察、显示）
# master = sentinel.discover_master('mymaster')
# print(master)
# 获取master的连接（用于数据操作），mymaster为sentinel中配置的不同redis集群标识
master = sentinel.master_for('mymaster', socket_timeout=0.5, password='123456', db=0)
master.set('name', 'bar')

# 获取slave的连接字符串（主要用于观察、显示）
# slave = sentinel.discover_slaves('mymaster')
# print(slave)
# 获取slave连接，从slave读取数据（默认是round-roubin）
slave = sentinel.slave_for('mymaster', socket_timeout=0.5, password='123456', db=0)
res = slave.get('name')
print(res)
```