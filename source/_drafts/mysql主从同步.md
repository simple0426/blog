---
title: mysql主从同步
tags:
categories:
---
# 复制功能优势
* 横向扩展解决方案：更新和写操作在主库，读操作在从库
* 数据安全性：从库可以停止同步、延迟同步、执行数据备份
* 离线数据分析
* 远距离数据分发

# 数据同步方案
* 默认，复制操作是单向的、异步的
* 同步解决方案：NDB集群：https://dev.mysql.com/doc/refman/5.6/en/mysql-cluster.html
* 半同步：主库事务的提交必须保证至少有一个从库也执行了相应操作：https://dev.mysql.com/doc/refman/5.6/en/replication-semisync.html
* 延迟复制【从库操作】：https://dev.mysql.com/doc/refman/5.6/en/replication-delayed.html
    - STOP SLAVE
    - CHANGE MASTER TO MASTER_DELAY = N;【seconds】
    - START SLAVE

# 复制日志的格式
* SBR（Statement Based Replication）：复制全部的sql语句，默认方式
* RBR（Row Based Replication）：复制变更的字段内容
* MBR（Mixed Based Replication）：前两种方式的变种组合
* GUIDs ：基于GUID的复制方式，不必直接基于binlog文件和文件中的position：https://dev.mysql.com/doc/refman/5.6/en/replication-gtids.html

# 主从复制的角色定位
* 主：只能开启binlog功能，并将全部事件（不能指定事件类型）记录在binlog中
* 从：
    - 记录已经在从库上读取并执行的binlog文件名和文件内的偏移信息（position）
    - 可以决定只执行binlog的部分内容【只同步部分库表，或不同步部分库表】
    - 可以决定：断开、重连、恢复同步动作


# 复制原理和细节：https://dev.mysql.com/doc/refman/5.6/en/replication-notes.html
# 管理和监控：https://dev.mysql.com/doc/refman/5.6/en/replication-administration.html
# 复制选项和参数：https://dev.mysql.com/doc/refman/5.6/en/replication-options.html

# 操作步骤
参考：https://dev.mysql.com/doc/refman/5.6/en/replication-howto.html
## 主配置【主】
>配置后需要重启server

* server-id=1
* log-bin=mysql-bin
* skip_networking=0 【默认已开启网络连接，这是一个只读变量，只能在配置文件设置】
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
* 在解锁【会话终端1中unlock tables或直接退出会话终端1】

### 逻辑导出-一键方式
>mysqldump自动处理锁表解锁、显示同步信息（导出的sql文件中）等

mysqldump --all-databases --master-data > dbdump.db

### 物理导出
* 适用于数据量较多的数据库
* 参考：https://dev.mysql.com/doc/refman/5.6/en/replication-howto-rawdata.html

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
* SHOW SLAVE STATUS\G
    - Slave_IO_Running：io线程是否在运行
    - Slave_SQL_Running：sql线程是否在运行
    - Seconds_Behind_Maste：主从延迟

# 从库信息
## 包含的文件
* relay log：由slave IO线程写入，包含从主库读取的binlog
* master info：包含连接主库的信息
* relay log info：从库relay log的执行情况

# 复制解决方案：https://dev.mysql.com/doc/refman/5.6/en/replication-solutions.html
## 备份只读库的设置
* 锁表： FLUSH TABLES WITH READ LOCK;
* 全局只读：SET GLOBAL read_only = ON;
* 备份：mysqldump
* 全局读解锁：SET GLOBAL read_only = OFF;
* 解锁表：UNLOCK TABLES;

## 只复制某些库表或忽略某些库表【只适用于基于row格式的binlog】
* Replicate_Do_DB
* Replicate_Ignore_DB
* Replicate_Do_Table
* Replicate_Ignore_Table
* Replicate_Wild_Do_Table
* Replicate_Wild_Ignore_Table

## 1主多从主故障处理
* 可以提升为主的从机要求：配置log-bin，没有配置log_slave_updates【作为从机时，要写binlog只能开启log_slave_updates】
* slave1【提升为master】
    - STOP SLAVE
    -  RESET MASTER
* slave2、slave3【其他从机】
    - STOP SLAVE
    - CHANGE MASTER TO MASTER_HOST='Slave1'【user, password, port】
    - START SLAVE 


# 参考信息
https://dev.mysql.com/doc/refman/5.6/en/replication.html