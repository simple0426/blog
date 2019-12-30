---
title: mongodb入门与管理
tags:
  - 安装
  - 用户权限
  - 管理
  - numa
categories:
  - mongodb
date: 2019-11-21 20:59:40
---
# 简介
## 存储引擎
* 3.2之后增加存储引擎WiredTiger
* 4.2之后移除对MMAPv1引擎的支持
* 若要将数据从MMAPv1迁移到WiredTiger，参考官方文档
* 数据目录下包含的文件必须和存储引擎匹配，否则无法启动
* WiredTiger存储引擎默认支持document级别锁（类似关系型数据库的行级锁）

## 软件版本
由于mongodb的不同版本的区别较大，需要确认不同操作系统版本支持的软件版本

## 数据可靠性
* Journal：使用Journal，即预写式日志保证数据可靠性
* 副本集：在4.0之后的版本，使用WiredTiger引擎、开启主从复制时不能关闭日志功能（storage.journal.enabled: false）

# 安装
## 安装要求
* 时钟同步
    - 目的：为了保证mongodb组件或集群成员之间保证正常工作，应该开启ntp服务以同步主机时间
    - 错误示例：【waited 189s for distributed lock configUpgrade for upgrading config database to new format v5】
* 设置文件描述符：ulimit
* 禁用Transparent Huge Pages
* 禁用NUMA选项
* 关闭selinux

## 安装组件
主要是安装mongodb-org，它包含如下的组件：
- mongodb-org-server：mongodb的服务端主进程__mongod__
- mongodb-org-mongos：mongodb的分片功能主进程__mongos__
- mongodb-org-shell：mongodb的客户端程序__mongo__
- mongodb-org-tools：mongodb的工具软件，比如mongodump,mongorestore, mongostat等

## centos安装
* 设置yum源
```
[mongodb-org-4.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.2/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.2.asc
```
* yum源更新：yum makecache
* 软件安装：sudo yum install -y mongodb-org

## ubuntu安装
* 设置软件源
```
wget -qO - https://www.mongodb.org/static/pgp/server-4.2.asc | sudo apt-key add -
echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/4.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.2.list
sudo apt-get update
```
* 软件安装：sudo apt-get install -y mongodb-org
* 避免自动升级
```
echo "mongodb-org hold" | sudo dpkg --set-selections
echo "mongodb-org-server hold" | sudo dpkg --set-selections
echo "mongodb-org-shell hold" | sudo dpkg --set-selections
echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
echo "mongodb-org-tools hold" | sudo dpkg --set-selections
```

## tar包安装
* 依赖安装：sudo yum install libcurl openssl
* [软件下载及解压](https://www.mongodb.com/download-center/community?jmp=docs)
* 编译安装（根据内置的参考文档）
* 将二进制文件路径添加到PATH
* 建立用户、日志目录、数据目录(对目录授权：mongod程序必须 有dbPath目录的读写权限)
* 编辑配置文件mongod.conf
* 启动服务：/usr/bin/mongod --config /etc/mongod.conf

# 服务控制
* 目录设置(mongod.conf)：
    - 数据目录：storage.dbPath
    - 日志目录：systemLog.path
* 启停：service mongod start/stop/restart
* 客户端连接：mongo

# 结构操作
## 库操作
* db：显示当前数据库
* show dbs：查看所有数据库
* use db：切换数据库（不存在则创建）
* db.dropDatabase()：删除当前库

## 集合操作
* 查看库的所有集合： show collections
* 创建集合：
    - 单独创建一个资源受限的空集合，超过限制则删除最早的数据
    - 语法范例：db.createCollection("profile",{size: 1024, capped: true, max: 5})
        + size：使用空间大小限制
        + capped：是否开显示
        + max：最多保存多少条数据
* 删除集合：db.collection.drop()
* 重命名集合：db.collection.renameCollection("new_name")

# 数据操作
## 增加
>集合不存在则自动创建

* 插入数据：db.collection.insert()
```
db.inventory.insert({"item": "test", "qty": 11, "status": "B", "size": {"h": 12, "w": 2, "uom": "cm"}, "tags": ["orange"]})
```
* 批量插入数据：db.collection.insertMany()
```
db.inventory.insertMany([
   { item: "journal", qty: 25, status: "A", size: { h: 14, w: 21, uom: "cm" }, tags: [ "blank", "red" ] },
   { item: "notebook", qty: 50, status: "A", size: { h: 8.5, w: 11, uom: "in" }, tags: [ "red", "blank" ] },
   { item: "paper", qty: 10, status: "D", size: { h: 8.5, w: 11, uom: "in" }, tags: [ "red", "blank", "plain" ] },
   { item: "planner", qty: 0, status: "D", size: { h: 22.85, w: 30, uom: "cm" }, tags: [ "blank", "red" ] },
   { item: "postcard", qty: 45, status: "A", size: { h: 10, w: 15.25, uom: "cm" }, tags: [ "blue" ] }
]);
// MongoDB adds an _id field with an ObjectId value if the field is not present in the document
```

## 删除
* 语法：db.collection.remove()
* db.inventory.remove({"item": "test"})

## 修改
* 语法：db.collection.update( [查询条件]，[更新内容]，[不存在时是否添加(默认false)]，[是否更新多行(默认false,只更新第一行)])
* 范例：
    + 内容变更：$set：db.inventory.update({"status": "A"}, {$set: {"qty": 35}},false,true)
    + 数值增减：$inc：db.inventory.update({"status": "A"}, {$inc: {"qty": -34}})

## 查询
* 查询所有内容： db.collection.find() 
    - 格式化输出(pretty)：db.inventory.find({}).pretty()
* 精确查询：db.inventory.find( { "size.uom": "in" } )
* 指定返回字段(默认返回id)
    - 语法：db.collection.find([查询字段]， [返回字段(1：包含，0：排除)])
    - 范例：db.inventory.find({},{_id:0, item:1, status:1})【不返回id，返回status】

# [用户和权限](https://www.cnblogs.com/dbabd/p/10811523.html)
* mongodb默认没有启用用户访问控制，可以无限制的访问数据库
* 权限由指定的数据库资源(resource)和指定资源上的操作(action)组成
    - 资源：数据库、集合、集群
    - 操作：增、删、改、查（CRUD）
* mongodb通过角色对用户授予相应数据库资源的操作权限，每个角色中的权限可以显式指定，也可以继承其他角色的权限，也可以两者兼有
        + 在同一个数据库中，新建的用户可以继承其他用户的角色
        + 在admin库中，创建的角色可以继承其他任意数据库中角色的权限
* mongodb中的角色分类
    - 系统内置角色
    - 用户自定义角色

## 系统内置角色
* 数据库用户角色
    - read：读取非系统集合数据
    - readWrite：读写非系统集合数据
* 数据库管理角色
    - dbAdmin：执行某些管理任务(与schema相关、索引、收集统计信息)的权限，但不包含用户管理
    - userAdmin：创建、修改角色、用户的权限
    - dbOwner：对数据库的所有的权限：readWrite、userAdmin、userAdmin
* 集群管理角色
    - clusterMonitor：监控工具（MongoDB Cloud Manager、Ops Manager）有只读权限
    - hostManager：包含针对数据库服务器的监控和管理操作
    - clusterManager：包含针对集群的监控和管理操作
    - clusterAdmin：拥有集群的最高权限：clusterManager、clusterMonitor、hostManager、dropDatabase
* 备份恢复角色
    - backup：拥有备份mongodb数据的最小权限
    - restore：含有从数据文件恢复数据的权限
* 全体数据库级别角色：只存在于admin库，适用于除了config和local之外的所有库
    - readAnyDatabase：对所有库的只读权限
    - readWriteAnyDatabase：对所有库的读写权限
    - userAdminAnyDatabase：类似于userAdmin对所有库的用户管理权限
    - dbAdminAnyDatabase：类似于dbAdmin对所有库的管理权限
* 超级用户角色
    - 用户拥有以下角色时，可以对任意用户授予任意数据库任意权限
        + admin库的dbOwner的角色
        + admin库的userAdmin的角色
        + userAdminAnyDatabase
    - root角色拥有的权限：readWriteAnyDatabase、dbAdminAnyDatabase、userAdminAnyDatabase、clusterAdmin、restore和backup
* 内部角色：\__system

## 用户管理和访问控制
### 管理用户
* 创建管理用户：`db.createUser({user: "user_admin", pwd: "user_admin", roles: [{role: "userAdminAnyDatabase", db: "admin"}]})`
* 开启访问控制(重启服务)：security.authorization：enabled
* 授权登录
    - 交互终端内：use admin；db.auth('user_admin','user_admin')
    - 命令行模式：mongo --host 172.16.0.201 --port 27017 -u user_admin -p user_admin --authenticationDatabase admin

### 普通用户
* 查询数据库所有用户：db.getUsers()、show users
* 创建用户并添加角色：
```
db.createUser(
{
    user: "user_dba",
    pwd: "user_dba",
    roles: ["read"],
    customData: {info: "user for dba"}
})
```
* 更新用户角色：db.updateUser("user_dba", {roles:[{role: "read", db: "admin"},{role: "readWrite",db: "examples"}]})
* 查询用户角色信息：db.getUser("user_dba",{showPrivileges: true})
* 添加用户角色：db.grantRolesToUser("user_dba",[{role: "readWrite", db: "admin"}])
* 回收用户角色：db.revokeRolesFromUser("user_dba", [{role: "read", db: "admin"}])
* 变更用户密码：db.changeUserPassword("user_dba","new_passwd")
* 删除用户：db.dropUser("user_dba")

# 备份和恢复
* 用户授权：`db.grantRolesToUser("user_admin",[{role: "backup", db: "admin"},{role: "restore", db: "admin"}])`
* 备份数据：`mongodump --port 30001 -u user_admin -p user_admin --authenticationDatabase=admin -d test -o ./backup/`
* 导出数据：mongoexport
    - 命令参数
        + -f：指定要导出的字段
        + --type：指定导出格式，默认json格式，也可指定csv
        + -d：数据库  -c：集合
    - mongoexport --host=host.com --port=3717 -u root -p passwd --authenticationDatabase=admin -d fubu -c fubu_forum -f username,phone,qq,email,homepage,recently_posted,last_reply_time --type=csv -o ./fubu_forum.csv
* 恢复数据：`mongorestore --port 30001 -u user_admin -p user_admin --authenticationDatabase=admin -d test --dir=./backup/test`

# 操作日志
## 原理
MongoDB Profile  记录是直接存在系统 db 里的，记录位置 system.profile ，  
所以，我们只要查询这个 Collection 的记录就可以获取到我们的 Profile  记录了  
## 设置
* 日志设置状态查询：db.getProfilingStatus();
* 命令行设置
    - 只能在非mongos的主设置
    - 设置：db.setProfilingLevel(1,500); 
    - 设置解读：0代表关闭，1代表只记录slowlog，2代表记录所有操作，这里设置成了500，即500ms。
* 配置文件设置
    - 开关：operationProfiling.mode: [off/slowOp/all]
    - 阈值：operationProfiling.slowOpThresholdMs: N

## 查询
* 交互式命令行：db.system.profile.find({millis:{$gt:500}})
* shell命令行：echo "db.system.profile.find({millis:{\$gt:500}})"|mongo 10.86.0.150:27017

# 服务性能优化
* 连接池【客户端设置】
* 内存：默认使用50% of (RAM - 1 GB)或256 MB中的大的数值
* cpu：默认的线城池数量和cpu核心数相关
* swap：系统设置swap分区，并设置swap【低度使用swap】
    - cat /proc/sys/vm/swappiness
    - sysctl vm.swappiness=1
* 文件系统
    - 尽量使用xfs文件系统
    - 不建议使用NFS等远程文件系统
    - 使用的文件系统支持目录级别fsync()【HGFS、virtualbox的共享目录不支持此操作】
* 磁盘：建议使用RAID-10. RAID-5
* 存储方式：将数据、预写日志、日志、索引存放在不同磁盘以改善性能
* 数据压缩存储
    - snappy：默认压缩方式，压缩率较低，cpu使用也低
    - zlib：压缩率较高，cpu使用也高
    - zstd：4.2版本出现的压缩选项，在压缩率和cpu中折中选择

------
# NUMA使用
## 介绍
现在的机器上都是有多个CPU和多个内存块的。以前我们都是将内存块看成是一大块内存，所有CPU到这个共享内存的访问消息是一样的。这就是之前普 遍使用的SMP模型。  
但是随着处理器的增加，共享内存可能会导致内存访问冲突越来越厉害，且如果内存访问达到瓶颈的时候，性能就不能随之增加。NUMA（Non-Uniform Memory Access）就是这样的环境下引入的一个模型。  
 比如一台机器是有2个处理器，有4个内存块。我们将1个处理器和两个内存块合起来，称为一个NUMA node，这样这个机器就会有两个NUMA node。在物理分布上，NUMA node的处理器和内存块的物理距离更小，因此访问也更快。  
 比如这台机器会分左右两个处理器（cpu1, cpu2），在每个处理器两边放两个内存块(memory1.1, memory1.2, memory2.1,memory2.2)，这样NUMA node1的cpu1访问memory1.1和memory1.2就比访问memory2.1和memory2.2更快。所以使用NUMA的模式如果能尽量保证本node内的CPU只访问本node内的内存块，那这样的效率就是最高的。
## 缺点
当你的服务器还有内存的时候，发现它已经在开始使用swap了，甚至已经导致机器出现停滞的现象。这个就有可 能是由于numa的限制，如果一个进程限制它只能使用自己的numa节点的内存，那么当自身numa node内存使用光之后，就不会去使用其他numa node的内存了，会开始使用swap，甚至更糟的情况，机器没有设置swap的时候，可能会直接死机！所以你可以使用numactl --interleave=all来取消numa node的限制。
## 使用建议
* 如果你的程序是会占用大规模内存的，你大多应该选择关闭numa node的限制。因为这个时候你的程序很有几率会碰到numa陷阱。  
* 如果你的程序并不占用大内存，而是要求更快的程序运行时间。你大多应该选择限制只访问本numa node的方法来进行处理  

## mongodb使用
mongod="numactl --interleave=all /usr/local/mongod/bin/mongod"
