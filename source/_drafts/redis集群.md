---
title: redis集群
tags:
categories:
---
# 集群介绍
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
* 集群数据分片：
    - 集群共有16384个hash槽，分布在集群的所有节点上
    - 每个key在逻辑上都是hash槽的一部分
    - 在给定一个key时，通过对key的CRC16相对于16384取模来确定key要存储的hash槽
    - 只需来回迁移hash槽到不同节点，就可以在线添加或删除节点
    - 集群支持在一个命令（整个事务、lua脚本执行）执行多键操作，只需要所有的键都在同一个hash槽中
    - 用户可以使用hash标签将多个键存储在同一个hash槽中
* 集群支持主从复制功能
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

# 内置脚本创建集群
utils/create-cluster/create-cluster start/create/stop
# 手动创建集群-官方参考
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

# 手动创建集群-脚本
* 解压redis源码，并将src目录绝对路径添加到PATH变量中
* 安装redis gem
* 执行创建脚本creat.sh

```
# create.sh
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

# 集群使用
* 客户端支持：
    - 程序语言支持：https://redis.io/topics/cluster-tutorial#playing-with-the-cluster
    - redis-cli支持(-c参数)：例如：redis-cli -c -p 7000
