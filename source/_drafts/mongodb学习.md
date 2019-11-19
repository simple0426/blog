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
