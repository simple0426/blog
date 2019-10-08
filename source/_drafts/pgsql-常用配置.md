---
title: pgsql-常用配置
tags:
categories:
---
# checkpoint
checkpoint会把所有的脏数据flush到磁盘【sql命令行执行：checkpoint】

* checkpoint_timeout：两次checkpoint间隔时长
* checkpoint_segments:：两次checkpoint间隔最大的xlog日志文件数量

# archive
xlog日志归档

* archive_mode：是否开启归档
* archive_command：将xlog拷贝到一个地方的命令

# 流复制
* max_wal_senders：master上执行流复制协议的wal_sender数量
* wal_keep_segments：xlog目录中最多容纳多少个wal日志文件，超过了则删掉最初的几个。（一个日志文件16M）
* hot_standby：允许standy节点执行查询

# 日志
>日志的默认位置都是在PGDATA目录下

* pg_log：这个日志一般是记录服务器与DB的状态，比如各种Error信息，定位慢查询SQL，数据库的启动关闭信息，发生checkpoint过于频繁等的告警信 息，诸如此类。
* pg_xlog：是pgsql的wal信息，也就是事务日志，这些日志会在定时回滚恢复(PITR)，流复制(Replication Stream)以及归档时能被用到，不能随意删除或移动
* pg_clog：是事务的元数据，这个日志告诉我们哪些事务完成了，哪些没完成。这个日志文件一般非常小，但是很重要。

## pg_log
* log_destination = 'csvlog'
* logging_collector = on
* 【慢日志】log_min_duration_statement = 200：只记录大于200ms的sql语句；0为记录所有语句，-1关闭记录功能。
