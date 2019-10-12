---
title: pgsql-流复制与高可用
tags:
categories:
---
# [前置要求][warm-standy]
* 表空间：如果使用表空间，因为表空间使用绝对路径的原因，需要主备可以建立相同的路径
* 硬件方面：主备的硬件架构需要一致，比如都是32或64位系统
* 软件版本：主备大版本要尽可能保持一致，备机小版本高于主是可以的，这是因为高版本可以读取低版本的wal文件

# 备机操作原理
* 重放wal的顺序
    - 归档目录中的文件
    - pg_xlog中的文件
    - 连接主机获取的wal
* 备机转换为正常的可读写主机
    - 使用命令：pg_ctl promote
    - 触发器文件：trigger_file

# 主机配置
* 归档目录可以被备机访问【如scp】，或直接在备机复制同样的归档文件
* 流复制设置：
    - 建立复制权限用户
        + sql---》create user repl replication password 'repl-123456';
        + pg_hba.conf ---》【host    replication     repl            172.16.0.0/24           password】
    - 其他配置：
        -   开启流复制线程【根据备机数量调整大小】：max_wal_senders 
        -   保持主备一致的配置：hot_standby = on
    - 【可选功能】 max_replication_slots

# 备机配置
* 【高可用】保持和主机相同的配置，如：归档、连接、授权等
* 从主机恢复一个基础备份：pg_basebackup -h 172.16.0.215 -U 'repl' -D data/ -RP
* 从主机复制归档文件：rsync -az -e ssh postgre@172.16.0.215:/usr/local/pgsql/archive archive
* 创建恢复配置文件 recovery.conf【pg_basebackup自动创建】
    - standby_mode = 'on'
    - primary_conninfo = 'host=192.168.1.50 port=5432 user=foo password=foopass'
    - restore_command = 'cp /path/to/archive/%f %p'
    - 【从机可读】 hot_standby = 'on'（postgresql.conf）
    - 【多备机故障切换】recovery_target_timeline=‘latest’
* archive_mode设置参考：
    - 在非恢复模式时，on和always作用一样，都会产生归档文件
    - 在恢复模式时，on并不会起作用（提升为主时才起作用）；always只有则会产生自己的归档文件【处理复杂，不建议使用】

# 状态监控
* 【主】pg_current_xlog_location：产生的wal记录
* 【备】pg_last_xlog_receive_location：已接收但未执行的wal记录
* 【主】pg_stat_replication：wal发送进程列表
    - sent_location与pg_current_xlog_location的差异表明主处于高负载状态
    - sent_location与pg_last_xlog_receive_location的差异表明主备的网络处于高负载，或从处于高负载状态

# 备升级为主【故障切换】
* 切换注意事项：需要保证同一时间只有一个主对外提供读写服务，通行的方案是heartbeat、vip偏移
* 切换方式
    - 方式1：设置 trigger_file（recovery.conf）
    - 方式2：使用 pg_ctl promote

# 其他复制技术
## 复制槽
* 作用：保护wal文件避免被轮转或删除，与wal_keep_segments、archive_command作用相同
* 优势：相比其他两种凡是，它只保持最小量的wal文件

## 级联复制
* max_wal_senders
*  hot_standby
*  host-based authentication

## 同步复制
* 原理与限制条件
    - 主无需从库返回确认信息的情况
        -  只读事务、事务回滚
        -  顶层的事务需要备库返回信息、而子事务不需要
        -  长时间运行的操作，包括数据加载、建立索引
    - 主需要从库返回确认信息：两阶段的操作（prepare、commit）
    - 因为主库必须知道从库的状态信息，主库和从库必须直连，所以级联不能使用同步复制
    - 主服务器必须等待所有同步服务器返回事务确认信息，否则事务永远不会完成；但可以定义同步、半同步、异步服务器减弱这种影响（synchronous_standby_names）
    - 在事务等待返回信息时创建备用服务器，应当在运行pg_start_backup和pg_stop_backup的同一个会话中设置synchronous_commit = off，否则事务提交会一直等待新的备用服务器出现才会完成
* 配置
    - synchronous_standby_names：从库名称信息，由application_name（recover.conf）组成，逗号分隔
        + 范例：synchronous_standby_names = '2 (s1, s2, s3)'
        + 解释：如果有s1, s2, s3 and s4 4个备用服务器，则s1，s2是主的同步服务器，s3是备主同步服务器（s1或s2出故障时被提升为主同步），s4为异步服务器【无需设置，默认为异步】
    - synchronous_commit：数据持久化级别【也可以在整个数据库级别、连接、事务级别指定数据持久化级别？？？】
        + on：默认设置，从库写一大块wal数据到磁盘
        +  remote_write：从库接收记录，并将内容写入操作系统，但不是持久化到磁盘
        +  remote_apply：从库已经应用接收的事务提交，但没有到数据写入阶段

# 高可用方案
* 共享存储：NAS
* 块设备复制（文件系统级别）：DRBD
* 事务日志传送：文件级别的日志传送、流复制
* 基于触发器的主备复制：Slony-I 
* 基于语句复制的中间件：Pgpool-II
* 异步多主复制：Bucardo 
* 同步多主复制：无
* 商业解决方案

## 其他解决方案
* 数据分区（Data Partitioning）：数据拆分后分布式存储，需要使用时再组合返回给终端
* 并行查询【提高查询速度】
    - 原理：将单个sql查询语句拆分成多个部分，分别交给不同后端处理，多个后端的查询结果经过聚合后返回给客户端
    - 实现：Pgpool-II、PL/Proxy

[warm-standy]: https://www.postgresql.org/docs/9.6/warm-standby.html
