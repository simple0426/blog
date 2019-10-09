---
title: pgsql-备份和恢复
tags:
categories:
---
# sql备份
* 可以远程执行备份
* 可以备份全部库、库、表、表结构、数据
* 备份的数据可以在当前版本或新版本数据库恢复
* 备份的数据可以同时在32或64系统上使用
* 备份的数据是备份开始的一个快照，他不会阻塞其他操作，但例外是会添加一个排它锁【如alter table】

## 命令
* pg_dump：只备份单个库
    - 默认备份为文本格式数据，可以使用psql命令恢复
        + 备份：pg_dump dbname > dumpfile
        + 恢复【需要先创建数据库】：psql dbname < dumpfile
    - 非文本格式【custom、directory、tar】使用pg_restore命令恢复
* pg_dumpall：备份全部的库、用户信息、表空间信息
    - 单独导出全局信息【用户、表空间】：pg_dumpall --globals-only
* pg_restore：恢复数据
    - 选项
        + -a：只恢复库数据
        + -s：只恢复数据库结构
        + -t：只恢复指定表
        + -d：连接的数据库
    - 范例
        + 恢复表结构 pg_restore -s -t xuji -d test xuji.sql
        + 恢复表数据 pg_restore -a -t xuji -d test xuji.db
        + 恢复表  pg_restore -t xuji -d test xuji.all

## pg_dump命令
### 命令选项
* -F：指定输出备份格式
* -a：只导出库数据
* -s：只导出数据库结构
* -t：备份指定表
* -T：不备份特定表
* -C：包含创建数据库命令
* -f：指定备份输出文件或目录
* -d：需要备份的数据库

### 备份范例
* 备份数据库 pg_dump -Fc test > test.all
* 备份表数据 pg_dump -Fc -a -t xuji test > xuji.db
* 备份表结构 pg_dump -Fc -s -t xuji test > xuji.sql

## 大数据量处理
* 使用压缩
    - pg_dump dbname | gzip > filename.gz
    - gunzip -c filename.gz | psql dbname
* 使用切分
    - pg_dump dbname | split -b 1m - filename
    - cat filename* | psql dbname
* 使用pg_dump自定义备份格式
    - pg_dump -Fc dbname > filename
    - pg_dump -Fc dbname > filename
* 并行处理
    - pg_dump -j num -F d -f out.dir dbname
    - pg_restore -j 

# 文件系统备份
## 限制条件
* 在开始备份或恢复前，需要关闭服务【只是停止对外服务时不可行的，因为无法产生原子快照或服务的内部缓冲机制】
* 无法单独恢复表【表文件和事务数据单独存储，只使用表文件和事务状态日志(pg_clog)则会造成其他表不可用】，文件级别备份只能用于完整备份和集群级别的恢复

## 不停机处理
* 获取数据目录的一致性"快照"；
    - 数据目录和WAL日志在不同磁盘，或表空间文件在不同文件系统时，获取一致性快照则比较困难
    - 可以使用归档备份【continuous archiving base backup】来获取一致性的数据
* 由于此"快照"是在服务运行时的数据，当此数据目录快照在执行恢复时会认为服务没有正常关闭，服务会应用上次checkpoint之后WAL日志【即，此时数据恢复也需要pg_xlog目录下的wal日志】；
* 在获取快照前，执行CHECKPOINT可以减少恢复时间。

# 持续归档和基于时间点恢复
## 归档设置
* 参数设置【postgresql.conf】
    - wal_level = replica
    - archive_mode = on
    - archive_command = 'test ! -f /mnt/server/archivedir/%f && cp %p /mnt/server/archivedir/%f'
* 建立归档目录，并设置目录权限为pg server运行用户
* 重启数据库
* 测试归档是否正常
    - checkpoint;
    - select pg_switch_xlog();

## 基础备份
* 方式1：使用pg_basebackup命令
* 方式2：使用API手动处理
    - 开启归档设置
    - 打开强制检查点【保持会话连接】：SELECT pg_start_backup('label', false, false);
    - 备份数据目录，需要排除的文件和目录如下
        + postmaster.pid、postmaster.opts【主配置】
        + recovery.conf、recovery.done【主配置】
        + pg_log
        + pg_xlog
        + pg_replslot
    - 关闭强制检查点：SELECT * FROM pg_stop_backup(false);
    - 【可选】：删除检查点之前的归档文件

### 备份实践
* 备份数据目录：tar czf /home/postgres/backup/data_$(date +%F).tar.gz     data/ --exclude=pg_xlog >/dev/null 2>&1
* 跨主机备份：rsync -C -a --delete -e ssh --exclude pg_log --exclude pg_xlog --exclude recovery.conf --exclude recovery.done data/ 10.150.10.41:~/db0/data/
* 删除无用归档文件： find ./ -mtime +1 -type f|xargs -i rm -f {} \;
* 跨主机同步归档文件：rsync -C -a --delete -e ssh archive/ 10.150.10.41:/home/postgres/db0/archive/
