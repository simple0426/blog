---
title: mongodb副本集部署
tags:
categories:
---
# 架构
* 主机：172.16.0.201、172.16.0.215、172.16.0.205
* 201已建立用户及其权限
    - readWriteAnyDatabase
    - clusterAdmin
    - userAdminAnyDatabase
    - dbAdminAnyDatabase

# 配置文件
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

# 集群配置
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

# 集群操作命令
* 查看状态：rs.status()
* 查看配置：rs.conf()
* 查看从机状态：db.printSlaveReplicationInfo();
* 查看复制状态：db.printReplicationInfo();

# 从机读设置
* 默认，读写都在primary节点操作，如果在从机操作，则会报错："not master and slaveOk=false"
* 从机客户端连接中设置可读：rs.slaveOk()
* 主机客户端设置读取策略：
    - mongo连接级别：db.getMongo().setReadPref('secondary')
    - 连接内的指针级别：db.inventories.find().readPref('secondary')
