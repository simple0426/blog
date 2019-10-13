---
title: pgsql-归档和流复制
tags:
  - 归档
  - 流复制
  - 高可用
categories:
  - pgsql
date: 2019-10-13 17:45:14
---

# 文件级别备份
## 限制条件
* 在开始备份或恢复前，需要关闭服务【只是停止对外服务是不可行的，因为无法产生原子快照或服务的内部缓冲机制】
* 无法单独恢复表【表文件和事务数据单独存储，只使用表文件和事务状态日志(pg_clog)则会造成其他表不可用】，文件级别备份只能用于完整备份和集群级别的恢复

## 不停机处理
* 获取数据目录的一致性"快照"；
    - 数据目录和WAL日志在不同磁盘，或表空间文件在不同文件系统时，获取一致性快照则比较困难
    - 可以使用归档备份【continuous archiving base backup】来获取一致性的数据
* 由于此"快照"是在服务运行时的数据，当此数据目录快照在执行恢复时会认为服务没有正常关闭，服务会应用上次checkpoint之后WAL日志【即，此时数据恢复也需要pg_xlog目录下的wal日志】；
* 在获取快照前，执行CHECKPOINT可以减少恢复时间。

# [归档备份和恢复][continuous-archiving]
## 注意事项
* wal文件不包含索引信息，所以索引需要手动处理
* 基础备份进行中，如果有 CREATE DATABASE执行会出现意料之外的事
* 创建表空间时，使用的是字面意思的绝对路径，在跨主机恢复会有问题

## 归档设置
* 参数设置【postgresql.conf】
    - wal_level = replica
    - archive_mode = on
    - archive_command = 'test ! -f /mnt/server/archivedir/%f && cp %p /mnt/server/archivedir/%f'
    - 【pg_baseback命令使用】max_wal_senders = 5
    - 【pg_baseback命令使用】wal_keep_segments = 10
* 【pg_baseback命令使用】添加复制用户（pg_hba.conf） 
    - 【local   replication     postgres                                trust】
* 建立归档目录，并设置目录权限为pg server运行用户
* 重启数据库
* 测试归档是否正常
    - checkpoint;
    - select pg_switch_xlog();

## 基础备份
* 方式1：使用pg_basebackup命令
* 方式2：使用sql命令行下的API手动处理
    - 开启归档设置
    - 打开强制检查点【保持会话连接】：SELECT pg_start_backup('label', false, false);
    - 备份数据目录，并排除如下文件和目录
        + postmaster.pid、postmaster.opts
        + recovery.conf、recovery.done
        + pg_log
        + pg_xlog
        + pg_replslot
    - 关闭强制检查点【同一会话】：SELECT * FROM pg_stop_backup(false);
    - 【可选】：删除检查点之前的归档文件

### 手动模式备份实践
* 备份数据目录：tar czf /home/postgres/backup/data_$(date +%F).tar.gz     data/ --exclude=pg_xlog >/dev/null 2>&1
* 跨主机同步数据目录：rsync -C -a --delete -e ssh --exclude pg_log --exclude pg_xlog --exclude recovery.conf --exclude recovery.done data/ 10.150.10.41:~/db0/data/
* 跨主机同步归档目录：rsync -C -a --delete -e ssh archive/ 10.150.10.41:/home/postgres/db0/archive/

## 恢复数据
* 停止服务
* 保留并转储pg_xlog目录内容，因为它们可能包含系统宕机时还没有归档的日志
* 删除数据目录及表空间根目录下的内容
* 从基础备份中恢复数据目录
* 删除基础备份中的pg_xlog的文件【因为它们可能比当前的数据陈旧】
* 将转储的pg_xlog文件复制到数据目录
* 在数据目录创建recovery.conf文件，并设置策略阻止普通用户连接并使用服务【pg_hba.conf或iptables】
* 【原理：恢复过程】：此时将使用上次基础备份以后的归档wal文件，及当前pg_xlog目录下的wal文件用于恢复数据
    - 归档目录不存在的内容将认为存在于pg_xlog中，这样就允许使用最近没有归档的wal文件
    - 归档目录中存在的内容优先于pg_xlog被应用
    - 恢复时将不会覆盖现有pg_xlog下的内容
* 恢复完成后，服务会自动将recovery.conf重命名为recovery.done
* 确认恢复完成后，开启对外服务

### recovery.conf
* 可以在安装目录的share目录下找到recovery.conf.sample作为范例
* 文件选项
    - 恢复命令【必选】：restore_command = 'cp /mnt/server/archivedir/%f %p'
    - 恢复截止时间：recovery_target_time (timestamp)
    - 是否包含截止点：recovery_target_inclusive (boolean)
        + 默认为true，包含截止点，即在截止点停止
        + false，不包含截止点，在截止点之前停止

# [流复制][warm-standy]
## 前置要求
* 表空间：如果使用表空间，因为表空间使用绝对路径的原因，需要主备可以建立相同的路径
* 硬件方面：主备的硬件架构需要一致，比如都是32或64位系统
* 软件版本：主备大版本要尽可能保持一致，备机小版本高于主是可以的，这是因为高版本可以读取低版本的wal文件

## 备机操作原理
* 重放wal的顺序
    - 归档目录中的文件
    - pg_xlog中的文件
    - 连接主机获取的wal
* 备机转换为正常的可读写主机
    - 使用命令：pg_ctl promote
    - 触发器文件：trigger_file

## 主机配置
* 归档目录可以被备机访问【如scp】，或直接在备机复制同样的归档文件
* 流复制设置：
    - 建立复制权限用户
        + sql---》create user repl replication password 'repl-123456';
        + pg_hba.conf ---》【host    replication     repl            172.16.0.0/24           password】
    - 其他配置：
        -   开启流复制线程【根据备机数量调整大小】：max_wal_senders 
        -   保持主备一致的配置：hot_standby = on
    - 【可选功能】 max_replication_slots

## 备机配置
* 【高可用】保持和主机相同的配置，如：归档、连接、授权等，其中archive_mode的设置参考如下
    - 在非恢复模式时，on和always作用一样，都会产生归档文件
    - 在恢复模式时，on并不会起作用（提升为主时才起作用）；always只有则会产生自己的归档文件【处理复杂，不建议使用】
* 从主机恢复一个基础备份：pg_basebackup -h 172.16.0.215 -U 'repl' -D data/ -RP
* 从主机复制归档文件：rsync -az -e ssh postgre@172.16.0.215:/usr/local/pgsql/archive archive
* 创建配置文件 recovery.conf【pg_basebackup自动创建】
    - standby_mode = 'on'
    - primary_conninfo = 'host=192.168.1.50 port=5432 user=foo password=foopass'
    - restore_command = 'cp /path/to/archive/%f %p'
    - 【从机可读设置】 hot_standby = 'on'（postgresql.conf）
    - 【多备机故障切换设置】recovery_target_timeline=‘latest’

## 状态监控
* 【主】pg_current_xlog_location：产生的wal记录
* 【备】pg_last_xlog_receive_location：已接收但未执行的wal记录
* 【主】pg_stat_replication：wal发送进程列表
    - sent_location与pg_current_xlog_location的差异表明主处于高负载状态
    - sent_location与pg_last_xlog_receive_location的差异表明主备的网络处于高负载，或从处于高负载状态

## 备升级为主
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
[continuous-archiving]: https://www.postgresql.org/docs/9.6/continuous-archiving.html