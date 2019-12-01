---
title: mongodb副本集与分片
tags:
  - 副本集
  - 分片
categories:
  - mongodb
date: 2019-12-01 18:55:08
---

# 副本集部署
## 架构
* 主机：172.16.0.201、172.16.0.215、172.16.0.205
* 201已建立用户及其权限
    - readWriteAnyDatabase
    - clusterAdmin
    - userAdminAnyDatabase
    - dbAdminAnyDatabase

## 配置文件
```
net: # 网络设置
  port: 27017
  bindIp: 0.0.0.0
security: 
  authorization: enabled # 开启用户认证
  keyFile: /etc/mongo_key.txt #集群认证秘钥
replication:
  replSetName: "rs0" # 副本集名称设置
```

## 集群认证秘钥
* 产生：openssl rand -base64 32 -out mongo_key.txt
* 变更用户和权限
    - chmod 0600 mongo_key.txt
    - chown mongodb mongo_key.txt

## 集群配置
>登录其中一个节点(201)配置，配置后此节点即为主节点

```
rs.initiate( {
   _id : "rs0",
   members: [
      { _id: 0, host: "172.16.0.201:27017" },
      { _id: 1, host: "172.16.0.215:27017" },
      { _id: 2, host: "172.16.0.205:27017" }
   ]
})
```

## 集群操作命令
* 查看状态：rs.status()
* 查看配置：rs.conf()
* 查看从机状态：db.printSlaveReplicationInfo();
* 查看复制状态：db.printReplicationInfo();

## 从机读设置
* 默认，读写都在primary节点操作，如果在从机操作，则会报错："not master and slaveOk=false"
* 从机客户端连接中设置可读：rs.slaveOk()
* 主机客户端设置读取策略：
    - mongo连接级别：db.getMongo().setReadPref('secondary')
    - 连接内的指针级别：db.inventories.find().readPref('secondary')

# 分片集群
## 注意事项
* 此部署文档适用于MongoDB 4.2
* 由于从3.6开始，所有分片集群必须是副本集；所有低于3.6版本的分片想要升级，必须
    - 先将独立的分片集群转换为副本集
    - 后将副本集升级为分片副本集集群

## 集群架构
* 3节点数据副本集rs0【rs0/172.16.0.201:27017,172.16.0.215:27017,172.16.0.205:27017】
* 3节点数据副本集rs1【rs1/172.16.0.201:27027,172.16.0.215:27027,172.16.0.205:27027】
* 3节点配置副本集【configReplSet/172.16.0.201:29017,172.16.0.215:29017,172.16.0.205:29017】
* 单节点路由服务【172.16.0.201:30001】

## 架构部署预览
* [创建3节点数据副本集rs0，批量插入测试数据](#创建数据副本集)
* [创建配置副本集(configsrv)](#创建配置副本集)
* [创建路由服务(mongos)](#创建路由服务)
* [将数据副本集添加为分片节点](#将数据副本集添加为分片节点)
* 创建第二个分片节点，即副本集rs1
* [将期望的集合分片](#将期望的集合分片)

# 创建数据副本集
* 参靠[mongodb副本集部署](#副本集部署)
* 使用自定义目录方式启动服务
    - 创建目录：mkdir -p /data/mongodb/rs0/{data,conf,log}
    - 创建用户：useradd mongodb
    - 对目录授权：chown -R mongodb.mongodb /data/mongodb/rs0
* 设置副本集名称：replication.replSetName: "rs0"
* 命令行以指定用户启动服务：` chroot --userspec "mongodb:mongodb" "/" sh -c "/usr/bin/mongod -f /data/mongodb/rs0/conf/mongod.conf"`
* 批量插入测试数据
```
use test
var bulk = db.test_collection.initializeUnorderedBulkOp();
people = ["Marc", "Bill", "George", "Eliot", "Matt", "Trey", "Tracy", "Greg", "Steve", "Kristina", "Katie", "Jeff"];
for(var i=0; i<1000000; i++){
   user_id = i;
   name = people[Math.floor(Math.random()*people.length)];
   number = Math.floor(Math.random()*10001);
   bulk.insert( { "user_id":user_id, "name":name, "number":number });
}
bulk.execute();
```
* 将副本集转换为可分片模式
    - 连接集群从节点：关闭服务后、添加配置【sharding.clusterRole: shardsvr】、重启服务
    - 连接集群主节点：主节点置为从【rs.stepDown()】、添加配置【sharding.clusterRole: shardsvr】、重启服务

# 创建配置副本集
* 参靠[mongodb副本集部署](#副本集部署)
* 设置配置集群选项
    - replication.replSetName: "configReplSet"
    - sharding.clusterRole: configsvr
* 初始化配置集群
```
rs.reconfig( {
   _id : "configReplSet",
  configsvr: true,
   members: [
      { _id: 0, host: "172.16.0.201:29017" },
      { _id: 1, host: "172.16.0.215:29017" },
      { _id: 2, host: "172.16.0.205:29017" }
   ]
})
```

# 创建路由服务
- 配置选项：
  + sharding.configDB: configReplSet/172.16.0.201:29017,172.16.0.215:29017,172.16.0.205:29017
  + security.keyFile: /etc/mongo_key.txt
- 启动服务：`chroot --userspec "mongodb:mongodb" "/" sh -c "/usr/bin/mongos -f /data/mongodb/mongos/conf/mongos.conf"`

# 将数据副本集添加为分片节点
* 连接到mongos并授权登录
* 添加副本集为分片节点：sh.addShard( "rs0/172.16.0.201:27017,172.16.0.215:27017,172.16.0.205:27017" )
* 查看集群分片状态：sh.status()

# 将期望的集合分片
* 连接到mongos并授权登录
* 数据库开启分片：sh.enableSharding( "test" )
* 在非空集合上添加索引：use test; db.test_collection.createIndex( { number : 1 } )
* 对集合分片：use test; sh.shardCollection( "test.test_collection", { "number" : 1 } )
* 确认分片处于平衡状态
```
use test
db.stats()
db.printShardingStatus()
```

# 配置文件
## 数据节点mongod.conf
```
storage:
  dbPath: /data/mongodb/rs0/data
  journal:
    enabled: true
systemLog:
  destination: file
  logAppend: true
  path: /data/mongodb/rs0/log/mongod.log
net:
  port: 27017
  bindIp: 0.0.0.0
processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  fork: true
security:
  authorization: enabled
  keyFile: /etc/mongo_key.txt
replication:
  replSetName: "rs0"
sharding:
  clusterRole: shardsvr
```
## 配置节点configsrv.conf
```
storage:
  dbPath: /data/mongodb/configdb/data
  journal:
    enabled: true
systemLog:
  destination: file
  logAppend: true
  path: /data/mongodb/configdb/log/config.log
net:
  port: 29017
  bindIp: 0.0.0.0
processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  fork: true
security:
  authorization: enabled
  keyFile: /etc/mongo_key.txt
replication:
  replSetName: "configReplSet"
sharding:
  clusterRole: configsvr
```
## 路由服务mongos.conf
```
systemLog:
  destination: file
  logAppend: true
  path: /data/mongodb/mongos/log/mongos.log
net:
  port: 30001
  bindIp: 0.0.0.0
processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  fork: true
sharding:
  configDB: configReplSet/172.16.0.201:29017,172.16.0.215:29017,172.16.0.205:29017
security:
  keyFile: /etc/mongo_key.txt
```
