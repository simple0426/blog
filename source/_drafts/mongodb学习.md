---
title: mongodb学习
tags:
categories:
---
# 安装
安装包mongodb-org

- mongodb-org-server：包含mongodb的服务端主进程：mongod程序
- mongodb-org-mongos：包含mongodb的分片功能主进程：mongos
- mongodb-org-shell：包含mongodb的客户端程序：mongo
- mongodb-org-tools：包含mongodb的其他客户端程序，比如 mongoimport bsondump, mongodump, mongoexport, mongofiles, mongorestore, mongostat, and mongotop

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
* 建立用户、日志目录、数据目录(对目录授权)
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
    - 单独创建一个一个资源受限的集合(超过限制则删除最早的数据)：
    - 语法：db.createCollection("profile",{size: 1024, capped: true, max: 5})
        + size：使用空间大小限制
        + capped：是否开显示
        + max：最多保存多少条数据
* 删除集合：db.collection.drop()
* 重命名集合：db.collection.renameCollection("new_name")

# 数据操作
## 增加
* 插入数据：db.collection.insert()
```
db.inventory.insert({"item": "test", "qty": 11, "status": "B", "size": {"h": 12, "w": 2, "uom": "cm"}, "tags": ["orange"]})
```
* 批量插入数据(不存在则创建)：db.collection.insertMany()
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
* 语法：db.collection.update( 查询条件, 更新内容,不存在时是否添加(默认false),是否更新多行(默认false,只更新第一行))
* 更新内容与范例：
    + 设置字段内容：$set：db.inventory.update({"status": "A"}, {$set: {"qty": 35}},false,true)
    + 字段值增减：$inc：db.inventory.update({"status": "A"}, {$inc: {"qty": -34}})

## 查询
* 查询所有内容： db.collection.find() 
    - 格式化输出(pretty)：db.inventory.find({}).pretty()
* 精确查询：
    - db.inventory.find( { "size.uom": "in" } )
* 指定返回字段(默认返回id)
    - 语法：db.collection.find(【查询字段】, 【返回字段】)
        - 返回字段定义：值为1表示返回此字段，值为0表示排除此字段
        - 范例：db.inventory.find({},{_id:0, item:1, status:1})
