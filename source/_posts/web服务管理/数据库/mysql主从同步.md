---
title: mysql主从同步
tags:
  - 主从复制
categories:
  - mysql
date: 2019-09-29 16:28:50
---

# 复制简介
## 复制功能优势
* 横向扩展解决方案：更新和写操作在主库，读操作在从库
* 数据安全性：从库可以停止同步、延迟同步、执行数据备份
* 离线数据分析
* 远距离数据分发

## 复制方式
* 异步复制
    - 一般结构【默认】
        + 特点：单层结构、所有slave直接连接master
    - 级联复制【操作如下】
        + 主：开启binlog
        + 主的从：开启binlog、log_slave_updates
        + 从的从：正常配置
    - [延迟复制][replication-delayed]【操作如下】
        + STOP SLAVE
        + CHANGE MASTER TO MASTER_DELAY = N;【seconds】
        + START SLAVE
    - [双向复制][master-master-mysql]【官方无此说明】：正向、反向各做一次主从同步操作，配置文件如下：
        ```
        # server-A
        server-id=1
        log-bin="mysql-bin"
        log_slave_updates   = 1
        binlog-ignore-db=test
        binlog-ignore-db=information_schema
        replicate-ignore-db=test
        replicate-ignore-db=information_schema
        relay-log="mysql-relay-log"
        auto-increment-increment = 2
        auto-increment-offset = 1
        expire_logs_days    = 10
        max_binlog_size     = 100M
        
        # server-B
        server-id=2
        log-bin="mysql-bin"
        log_slave_updates   = 1
        binlog-ignore-db=test
        binlog-ignore-db=information_schema
        replicate-ignore-db=test
        replicate-ignore-db=information_schema
        relay-log="mysql-relay-log"
        auto-increment-increment = 2
        auto-increment-offset = 2
        expire_logs_days    = 10
        max_binlog_size     = 100M
        ```
        - 问题
            + 需要避免在两主上同时对一个表数据进行修改
            + 不能解决磁盘io压力
* 同步复制：[NDB集群][mysql-cluster]
* [半同步][replication-semisync]：主库事务的提交必须保证至少有一个从库也执行了相应操作【阿里云rds高可用即采用此种方式】

## binlog日志格式
* SBR（Statement Based Replication）：复制全部的sql语句，默认方式
    - 复制存储过程和触发器时，可能会有问题
    - binlog文件记录的内容少，备份和还原也更快
    - 记录原始的sql语句，可以用于安全审计
* RBR（Row Based Replication）：复制变更的字段内容
    - 可以精确记录表和字段是如何更改的，因此binlog文件记录的内容较多
    - 不能直观的看到sql执行历史，解析binlog时：【mysqlbinlog --base64-output=DECODE-ROWS --verbose】
* MBR（Mixed Based Replication）：前两种方式的变种组合
* [GUIDs][replication-gtids] ：基于GUID的复制方式，不必直接基于binlog文件和文件中的position

## 主从复制的角色定位
* 主：只能开启binlog功能，并将全部事件（不能指定事件类型）记录在binlog中
* 从：
    - 记录已经在从库上读取并执行的binlog文件名和文件内的偏移信息（position）
    - 可以决定只执行binlog的部分内容【只同步部分库表，或不同步部分库表】
    - 可以决定：断开、重连、恢复同步动作

## [复制原理和细节][replication-implementation-details]
* 主库配置log-bin后等待从机连接
* 从机使用change master命令配置连接和复制状态信息【存储在master.info文件】，使用start slave开启复制
* 从机的io线程和主机的binlog dump线程通信，传输binlog
* 从机的io线程将收到的binlog存入relay-log文件中
* 从机的sql线程读取relay-log中的日志，并在数据库中执行相应的sql操作【状态信息保存在relay-log.info】

# [操作步骤][replication-howto]
## 主配置【主】
>配置后需要重启server

* server-id=1
* log-bin=mysql-bin
* 以下为保持innodb事务一致性的设置：
    - innodb_flush_log_at_trx_commit=1
    - sync_binlog=1

## 从配置【从】
>配置后需要重启server

* server-id=2

## 建立复制账号【主】
grant replication slave on *.* to 'repl'@'172.16.0.215' identified by 'repl20190926';

## 导出数据【主】
* 为了保证数据一致性，在导出过程中需要锁表
* 在mysql的终端会话中锁表，退出该会话会导致锁表失效；因此需要，在一个会话终端中锁表，在另一个会话终端中导出数据

### 逻辑导出-手动方式
* 锁表【会话终端1】： FLUSH TABLES WITH READ LOCK；
* 导出数据【会话终端2】：mysqldump
* 查看同步信息【会话终端1】：SHOW MASTER STATUS;
* 解锁【会话终端1中unlock tables或直接退出会话终端1】

### 逻辑导出-一键方式
>mysqldump使用--master-data参数时，会自动处理锁表解锁、显示同步信息（导出的sql文件中）等

mysqldump --all-databases --master-data > dbdump.db

### [物理导出][replication-howto-rawdata]
适用于数据量较多的数据库

## 导入数据【从】
>从库已有同步时，则使用--skip-slave-start启动server，或stop slave停止同步

* 将导出的数据传送到从库所在主机
* 从库导入数据：mysql < all.sql 

## 使用命令同步【从】
* 连接主库

```
mysql> CHANGE MASTER TO
->     MASTER_HOST='master_host_name',
->     MASTER_USER='replication_user_name',
->     MASTER_PASSWORD='replication_password',
->     MASTER_LOG_FILE='recorded_log_file_name',
->     MASTER_LOG_POS=recorded_log_position;
```

* 开启同步：START SLAVE;

## 查看同步状态【从】
* SHOW SLAVE STATUS\G;
    - Slave_IO_State：io线程状态
    - Slave_IO_Running：io线程是否在运行
    - Slave_SQL_Running：sql线程是否在运行
    - Seconds_Behind_Maste：主从延迟

# [其他复制选项][replication-solutions]
## 防止从库写操作
* 从库配置read-only=yes参数【此时只允许从服务器线程或super权限用户执行更新操作】
* 复制忽略mysql、information_schema库，主从建立单独的应用连接用户，从库建立的用户只授予读权限（select）

## 过滤库表
* 数据库级别的设置(do-db、ignore-db)：会由于binlog格式SBR和RBR的不同产生不同效果
    - --replicate-do-db
    - --replicate-ignore-db
* 表级别的设置：不会因binlog格式不同产生不同效果，因此要尽量使用table级别设置
    - --replicate-do-table
    - --replicate-ignore-table
    - --replicate-wild-ignore-table
    - --replicate-wild-do-table

## 提升复制性能
* 级联复制
* 其他提升复制性能操作
    - 数据文件、binlog文件、relay-log文件放到不同物理磁盘
    - 将实例的不同数据库同步到不同从机
    - 如果从机不作为其他从机的主，可以注释log_slave_updates，避免从机写binlog文件

## 主故障切换
* 从提升为主的配置：配置log-bin，禁止log_slave_updates
    - 从机开启log-bin参数不会写binlog，同时配置log_slave_updates则会写binlog
    - 禁止log_slave_updates：当从机【slave-1】配置log_slave_updates，如果主故障，需要切换此从机【slave-1】为主机时，其他从机【slave-2】会在已接收过原master的binlog更新的同时，也接收新master【slave-1】的binlog更新，此时会出现重复操作
* 所有从机停止io线程【STOP SLAVE IO_THREAD】，并确认sql线程已经完全读取并执行relay-log中的信息【show proceslist】
* slave1【提升为master】
    - STOP SLAVE
    - RESET MASTER
* slave2、slave3【其他从机】
    - STOP SLAVE
    - CHANGE MASTER TO MASTER_HOST='Slave1'【user, password, port】
    - START SLAVE 

# 故障处理
* 确认master开启binlog【show master status】
* 确认master和slave的server-id不一样
* 确认slave的 Slave_IO_Running和Slave_SQL_Running状态正常【show slave status】
* 确认slave没有手动写入数据，造成数据不一致
* 跳过来自主服务器的下一个语句【事件：事务类的一条事务，非事务类的一条sql语句】
    - 语法：SET GLOBAL sql_slave_skip_counter = N;
    - N=1的情况：来自主服务器的下一个语句不使用AUTO_INCREMENT或LAST_INSERT_ID()
    - N=2的情况：使用AUTO_INCREMENT或LAST_INSERT_ID()时，此时它们从主服务器的二进制日志中取两个事件
* 配置跳过指定的错误号【比如：由于重复造成的不能入库】
    - 语法：slave-skip-errors=1032,1062,1007

---
[replication-howto]: https://dev.mysql.com/doc/refman/5.6/en/replication-howto.html
[replication-howto-rawdata]: https://dev.mysql.com/doc/refman/5.6/en/replication-howto-rawdata.html
[replication-solutions]: https://dev.mysql.com/doc/refman/5.6/en/replication-solutions.html
[replication-gtids]: https://dev.mysql.com/doc/refman/5.6/en/replication-gtids.html
[mysql-cluster]: https://dev.mysql.com/doc/refman/5.6/en/mysql-cluster.html
[replication-semisync]: https://dev.mysql.com/doc/refman/5.6/en/replication-semisync.html
[replication-delayed]: https://dev.mysql.com/doc/refman/5.6/en/replication-delayed.html
[replication-implementation-details]: https://dev.mysql.com/doc/refman/5.6/en/replication-implementation-details.html
[master-master-mysql]: https://dinfratechsource.com/2018/11/11/configure-master-master-mysql-database-replication/
