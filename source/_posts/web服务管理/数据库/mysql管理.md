---
title: mysql管理
tags:
  - 存储过程
  - mysqladmin
  - 锁表
  - 架构
  - 高可用
categories:
  - mysql
date: 2019-09-30 16:52:53
---

# 管理命令
## mysqladmin
shell命令行下服务端mysqld的管理工具，帮助信息：mysqladmin --help

## mysql内建命令
* system：调用系统命令
* help：帮助信息
* show `[full]`processlist;：进程列表
* show variables;：变量信息
* show `[func_name]` status;：状态和统计信息

# 使用优化
* 能用定长char的就不用varchar
* 查询时，尽量不要使用select \*，同时要用where条件匹配
* 尽量使用批量插入

# 锁表处理
## 情景1
* 现象与解释：mysql命令行执行时，sql挂起；使用show processlist查看，该sql线程状态为【waiting for table metadata lock】；这是由于该表有正在执行的其他长事务（sql），进而阻塞了同表的后续操作。
* 处理
    - 使用【show processlist】查看长事务的线程id
    - 使用kill命令杀死改线程

## 情景2
* 现象和解释：对该表进行操作时，客户端显示锁表（比如：java客户端中出现错误【Lock wait timeout exceeded; try restarting transaction】，但是通过show processlist命令看不到该表的任何操作；实际上该表存在未提交的事务，可以在
information_schema.innodb_trx中查看到。
* 处理
    - 使用【select * from information_schema.innodb_trx\G】查找相关表的线程id
    - 使用kill命令杀死该线程【由于是事务类型操作，杀死线程后，事务会回滚】

# 读写分离
* 应用程序自己实现读写路由
* 官方代理插件：mysql-proxy(已停止维护)--》MySQL Router【需要在应用中配置读写端口】
* 第三方插件：
    - Amoeba(已停止维护)--》Cobar(已停止维护)--》[MyCAT](https://github.com/MyCATApache/Mycat2)
    - [atlas][mysql-atlas]：360公司团队基于mysql-proxy进行的二次开发

# [高可用方案](https://www.cnblogs.com/robbinluobo/p/8294782.html)
## 数据同步+故障切换
* 数据同步
    - 异步
    -[半同步][replication-semisync]：主库事务的提交必须保证至少有一个从库也执行了相应操作【阿里云rds高可用即采用半同步】
* 故障切换
    - [MMM(Master-Master replication managerfor Mysql)](https://www.jianshu.com/p/7331779dbae8)
    - MHA（Master High Availability）
        + MySQL阿里云ecs不支持虚拟ip，所以无法使用keepalived、heartbeat类高可用软件
        + 需要添加多个多个节点之间的ssh免密登陆

## 共享存储
* SAN：数据库服务器和存储分离
* DRBD：DRBD是一种基于软件、基于网络的块复制存储解决方案，主要用于对服务器之间的磁盘、分区、逻辑卷等进行数据镜像，当用户将数据写入本地磁盘时，还会将数据发送到网络中另一台主机的磁盘上，这样的本地主机(主节点)与远程主机(备节点)的数据就可以保证实时同步。

## 分布式协议
* mysql cluster(官方)：使用NDB存储引擎的集群，因为各种问题，国内应用的不多
* group replication(官方)：mysql5.7出现的组复制功能
    - 单主模式下，组复制具有自动选主功能，每次只有一个 Server成员接受更新，其它成员只提供读服务。
    - 多主模式下，所有的Server 成员都可以同时接受更新，没有主从之分，成员角色是完全对等的。
* [Galera](https://blog.51cto.com/11912662/2155443)

[mysql-atlas]: https://blog.csdn.net/xiaoying5191/article/details/81112747#2-%E8%AF%BB%E5%86%99%E5%88%86%E7%A6%BB%E4%B8%AD%E9%97%B4%E4%BB%B6atlas
