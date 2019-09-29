---
title: mysql架构设计
tags:
categories:
---
# 一主多从的架构
* 1台master用于写操作
* 1-3台slave用于提供读的负载均衡
* 1台slave提供离线访问【如后台访问，数据分析等】
* 1台开启binlog实现增量备份与恢复

# 读写分离
* 应用程序自己实现读写路由
* 官方代理插件：mysql-proxy(已停止维护)--》MySQL Router【需要在应用中配置读写端口】
* 第三方插件：Amoeba(已停止维护)--》Cobar(已停止维护)--》[MyCAT](https://github.com/MyCATApache/Mycat2)
    - [cobar与mycat](https://blog.csdn.net/xuheng8600/article/details/79843808)

# 主库高可用
## 半同步
[半同步][replication-semisync]：主库事务的提交必须保证至少有一个从库也执行了相应操作【阿里云rds高可用即采用此种方式】

## 组复制(group replication)
mysql5.7支持

## 文件系统同步
官方zfs，或块同步软件DRBD

## 主故障自动切换
MHA（Master High Availability）
