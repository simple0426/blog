---
title: redis集群
tags:
  - 集群
categories:
  - redis
date: 2019-10-26 00:48:05
---

# [集群介绍](https://redis.io/topics/cluster-tutorial)
* redis3.0之后提供集群模式
* 集群功能
    - 自动将数据拆分到多个节点
    - 在一部分节点不可用时，依然可以进行数据操作
* 集群端口
    - 服务端口(client port，如6379)：集群的其他节点(key迁移)、客户端都可以访问到
    - 集群总线端口(client port+10000)：用于集群节点间的通信，如：失败检测、配置更新、故障切换授权等
* 集群与docker
    - 当前redis集群不支持nat网络环境和ip、port都重新映射的通用环境
    - 为了使docker兼容redis集群，docker应当使用host网络模式（--net=host）
* 数据分片：
    - 集群共有16384个hash槽，分布在集群的所有节点上
    - 每个key在逻辑上都是hash槽的一部分
    - 在给定一个key时，通过对key的CRC16相对于16384取模来确定key要存储的hash槽
    - 只需来回迁移hash槽到不同节点，就可以在线添加或删除节点
    - 集群支持在一个命令（整个事务、lua脚本执行）执行多键操作，只需要所有的键都在同一个hash槽中
    - 用户可以使用hash标签将多个键存储在同一个hash槽中
* 支持主从复制功能
* 数据一致性保证
    - redis集群不保证强的数据一致性：比如在数据刷新到磁盘前宕机、数据复制到从机前宕机等都会丢失部分数据
    - 可以通过如下措施增加一致性
        + 变更数据刷新策略（fsync），即数据写入磁盘后才向客户端返回确认信息，但这回造成性能损失
        + 使用WAIT命令后，启用同步复制功能
    -   网络隔离也会造成数据丢失
        +   产生网络隔离后，客户端Z1和至少一个主节点B位于其中一个网络，其他节点（A、A1、B1、C、C1）位于另一个网络
        +   在节点超时前(cluster-node-timeout)，Z1依然可以向B发送写操作；
        +   节点超时后，不能和其他主节点通信的主节点(B)会被认为是失败节点，不再接收写操作，集群也会将失败节点的从节点(B1)提升为主节点
        +   由于B在不能和从节点通信时依然接收写操作，当从节点B1提升为主节点时会丢失部分数据
* 集群配置参数
    - cluster-enabled：是否开启集群模式，集群模式和独立模式不能互相切换
    - cluster-config-file：集群状态文件，用户不可编辑
    - cluster-node-timeout：集群超时时间，超时则失败主停止写，从机执行故障切换
    - cluster-slave-validity-factor：【超时时间*衡量因子】之后从机执行故障切换，为了高可用，可以设置为0【从机一直尝试接管主机】
    - cluster-migration-barrier：主机应当保留的从机个数，超过的从机可以迁移到孤立主机上
    - cluster-require-full-coverage：数据不全时是否对外提供查询服务

# 创建集群
## 内置脚本方式
utils/create-cluster/create-cluster start/create/stop
## 手动创建方式
* 创建并启动多个redis实例(包含集群参数)
* 创建集群
    - 版本5之后已在redis-cli命令中内置集群功能，可以实现：创建集群、集群分片检查和重新分片等功能
        + 范例：redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 --cluster-replicas 1
        + 解释：使用create创建集群，并保证每个主一个从节点
    - 版本4和3需要使用ruby工具：redis-trib.rb
        + 安装 redis gem：
            * apt-get install rubygems
            * gem install redis
        + 使用ruby工具创建集群
            * ./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005

## 手动创建范例
* 解压redis源码，并将src目录绝对路径添加到PATH变量中
* 安装redis gem
* 执行创建脚本creat.sh

```
# create.sh
result="redis-trib.rb create --replicas 1 "
for port in `seq 7000 7005`
do
        mkdir ${HOME}/${port}
        cp redis.conf ${HOME}/${port}/${port}.conf
        sed -i 's/7000/'$port'/g' ${HOME}/${port}/${port}.conf
        sed -i 's%/home/redis%'${HOME}'%g' ${HOME}/${port}/${port}.conf
        echo $port
        redis-server ${HOME}/${port}/${port}.conf
        ip_port="127.0.0.1:"$port
        result=${result}\ ${ip_port}
done
echo "yes" |$result

# stop.sh
for port in `seq 7000 7005`
do
    cat ${HOME}/${port}/${port}.pid  |xargs -i kill {}
done
```
# 客户端支持
- 程序语言支持：https://redis.io/topics/cluster-tutorial#playing-with-the-cluster
- redis-cli支持(-c参数)：例如：redis-cli -c -p 7000

# 集群管理
* hash槽迁移：
    - 命令行(redis-trib)：redis-trib.rb reshard --from source_id --to dest_id --slots slot_num --yes ip:port
    - 参数解释
        + 连接的节点ip和端口需要放在最后
        + 源和目标节点id可以从【cluster nodes】中获取，源节点可以有多个(逗号分隔，all表示其他所有节点)
        + --from/--to：迁移的源和目标节点
        + --slots：迁移的槽数量
        + --yes：免交互确认
* 手动故障切换(主从状态切换)：cluster failover（在从节点执行）
* 添加主节点：
    - 命令行方式：redis-trib  add-node  new_host:new_port existing_host:existing_port
    - 交互式命令行：cluster meet ip port 
* 添加从节点：
    - 命令行方式：redis-trib  add-node --slave --master-id new_host:new_port existing_host:existing_port【不指定master-id时，默认将其随机添加到具有最少从节点的主节点】
    - 交互式命令行：cluster replicate master-id
        + 此种模式首先添加一个没有数据的空节点，然后连接到此节点后使用上述复制命令将其添加为从节点
        + 此命令也可以用于改变从机的主节点
* 删除节点：
    - 删除方式：
        + 命令行：redis-trib del-node host:port node_id
        + 交互式命令行：cluster forget node_id
    - 主从处理：
        + 从节点可以直接删除
        + 主节点没有数据时才可以删除

# 可用性预处理
为了提高系统可用性，事先在每个实体机上预留1~2个多余的从节点作为备份，当某个主节点或从节点挂掉时，多余的从节点可以自动的迁移(cluster-migration-barrier)过来以达到集群所有主节点至少有一个从节点。
## 少数主或从节点故障
* 预处理：配置备份飘逸参数cluster-migration-barrier
* 主节点故障：客户端写操作会报错；不会形成备份飘逸；8、9秒后，从节点提升为主，客户端可以进行写操作。
* 从节点故障：客户端无影响；会形成备份飘逸

## 少数主和它的从节点同时挂掉
* 预处理：配置cluster-require-full-coverage为no，集群出故障时可读不可写

## 多数主节点同时宕机
集群处于不可用状态

# 数据迁移
* redis支持迁移到集群的情况
    - 没有使用多键操作、事务、包含多键的lua脚本
    - 使用hash标签的多键、事务、包含多键的lua脚本
* 手动迁移步骤
    - 停止客户端连接。当前版本不能进行在线迁移
    - 产生aof文件【BGREWRITEAOF】
    - 拷贝aof文件到其他位置，停止旧实例
    - 创建包含N个主和0个从的集群【从机稍后添加，并确保使用aof做持久化化】
    - 停止所有集群节点，将迁移前产生的aof文件替换到集群中
    - 重启集群
    - 使用redis-trib fix让键值对自动迁移
    - 使用redis-trib check查看迁移结果
    - 重启客户端，连接到集群任意节点
    - 向当前集群添加从节点
* 5.0版本内置迁移命令：redis-cli --cluster import
    - 支持将一个运行中的实例数据迁移到一个预先搭建好的集群中（同时会删除源实例中的数据）
    - 当2.8版本作为源实例时，由于没有实现迁移连接缓存，迁移过程会很慢，解决方案是使用3.x版本实例加载源数据

# 故障处理
* 交互式迁移报错
    - 报错：[ERR] Calling MIGRATE: ERR Syntax error, try CLIENT (LIST | KILL | GETNAME | SETNAME | PAUSE | REPLY)
    -  原因：官方bug
    -  解决：安装低版本gem redis：gem install redis -v 3.3.5
* 节点状态不一致
    - 现象
        + 连接不同的节点，获取cluster nodes输出不一致
        + 使用./redis-trib [check|fix]命令得到如下报错：【 Nodes don't agree about configuration】，并显示个别slot处于importing状态
        + 由于状态不一致，所以不能进行slot迁移【不能使用reshard命令】
    - 解决：由于只是个别节点状态不一致，只需处理个别节点即可
    - 步骤
        + 手动设置异常节点的slot：for i in {5461..10922};do redis-cli -h 172.29.88.117 -p 7004 -c cluster setslot $i node ca4757c010e6b6c028006bb4b8c1cd0852ab3033;done
            * 5461...10922为状态异常的slot
            * -h/-p  连接异常节点
            * ca4757c0... 为这些slot的正确节点id【根绝大多数节点返回获得】
        + 修复异常节点状态（循环：每次修复一个slot）：for i in {5461..10922};do ./redis-trib.rb fix 172.29.88.117:7004;done
