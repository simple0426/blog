---
title: pgsql-安装与介绍
tags:
  - postgresql
  - pgsql
categories:
  - pgsql
date: 2019-10-13 16:52:13
---

# 依赖安装
>示例为ubuntu16.04下的软件包

* make
* gcc
* readline：libreadline6 libreadline6-dev【用于sql命令行提示】
* zlib：zlib1g zlib1g-dev【备份和恢复时压缩】
* libpython2.7 libpython2.7-dev libpython3.5 libpython3.5-dev【python语言支持】
* ssl：OpenSSL【连接加密】
* xml2：libxml2 libxml2-dev
* xslt：libxslt1.1  libxslt1-dev

# 软件安装
>软件版本：v9.6.15

* ./configure --with-python --with-openssl --with-libxml --with-libxslt
* make 
* sudo make install

# 链接库设置
* 动态设置【官方参考】
    - 【普通用户】LD_LIBRARY_PATH=/usr/local/pgsql/lib;export LD_LIBRARY_PATH
    - 【root】/sbin/ldconfig /usr/local/pgsql/lib
* 持久化设置：echo /usr/local/pgsql/lib >> /etc/ld.so.conf;ldconfig

# 环境变量设置
```
echo "PATH=/usr/local/pgsql/bin:\$PATH" >> /etc/profile
echo "export PATH" >> /etc/profile
source /etc/profile
```

# 初始化数据库
```
adduser postgres
mkdir /usr/local/pgsql/data
chown postgres /usr/local/pgsql/data
su - postgres
initdb -D /usr/local/pgsql/data --no-locale
pg_ctl -D /usr/local/pgsql/data -l logfile start
createdb test
psql test
```

# postgis安装
postgis是PostgreSQL的空间和地理对象
## [依赖安装](http://www.postgis.net/docs/postgis_installation.html#install_requirements)
* PostgreSQL9.4以上
* gcc
* make
* proj4：proj-bin
* geos：libgeos-c1v5 libgeos-dev 
* xml2：libxml2 libxml2-dev
* json-c：libjson-c2 libjson-c-dev
* gdal：libgdal20 gdal-bin libgdal-dev 
* llvm：llvm-6.0 llvm-6.0-dev 

## 软件安装
* ./configure --with-pgconfig=/usr/local/pgsql/bin/pg_config
* make
* sudo make install

## 开启postgis
* postgis：psql -d test -c "CREATE EXTENSION postgis;"
* postgis_topology:psql -d test -c "CREATE EXTENSION postgis_topology;"

# 主要命令
* createdb、createuser：创建库、用户
* dropdb、dropuser：删除库、用户
* initdb：初始化数据库
* pg_basebackup：创建数据目录的备份【主要用于归档中建立基础数据备份】
* pg_config：一般用于扩展插件编译时加载pgsql的header信息
* pg_ctl：数据库服务启停控制
* pg_dump：逻辑备份单个库表
* pg_dumpall：备份整个数据库实例的全部信息
* pg_restore：用于恢复非文本类型的备份
* postgres：数据库服务运行程序
* psql：pgsql的客户端

# 名词术语
## checkpoint
checkpoint(sql命令：checkpoint)会把所有的脏数据flush到磁盘，部分参数如下：

* checkpoint_timeout：两次checkpoint间隔时长
* checkpoint_segments:：两次checkpoint间隔最大的xlog日志文件数量

## archive
xlog(事务)日志归档，部分参数如下：

* archive_mode：是否开启归档
* archive_command：将xlog拷贝到一个地方的命令

## 流复制
* max_wal_senders：master上执行流复制协议的wal_sender数量
* wal_keep_segments：xlog目录中最多容纳多少个wal日志文件，超过了则删掉最初的几个。（一个日志文件16M）
* hot_standby：是否允许standy节点执行查询功能

## 日志
pgsql的各种日志的默认位置都是在PGDATA目录下，主要包含：

* pg_log：这个日志一般是记录服务器与DB的状态，比如各种Error信息，定位慢查询SQL，数据库的启动关闭信息，发生checkpoint过于频繁等的告警信 息，诸如此类，常见参数如下
    - log_destination = 'csvlog'
    - logging_collector = on
    - 【慢日志】log_min_duration_statement = 200：只记录大于200ms的sql语句；0为记录所有语句，-1关闭记录功能。
* pg_xlog：是pgsql的wal信息，也就是事务日志，这些日志会在定时回滚恢复(PITR)，流复制(Replication Stream)以及归档时能被用到，不能随意删除或移动
* pg_clog：事务的状态信息，这个日志告诉我们哪些事务完成了，哪些没完成。这个日志文件一般非常小，但是很重要。
