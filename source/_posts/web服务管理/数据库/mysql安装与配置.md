---
title: mysql安装与配置
tags:
  - 安装
  - my.cnf
categories:
  - mysql
date: 2019-09-24 16:39:59
---

# 官方参考
参考手册：https://dev.mysql.com/doc/refman/5.6/en/
# 源码安装
## [依赖安装](https://dev.mysql.com/doc/refman/5.6/en/source-installation-prerequisites.html)
* cmake
* make OR gmake
* GCC 4.2.1 or later
    - centos系列使用gcc*
* SSL library
* ncurses
    - ubuntu16：libncurses5-dev、libncurses5

## [预配置](https://dev.mysql.com/doc/refman/5.6/en/installing-source-distribution.html)
* groupadd mysql
* useradd -r -g mysql -s /bin/false mysql
* mkdir /application

## 软件解压
tar zxvf mysql-VERSION.tar.gz
## 编译配置
* cd mysql-VERSION
* 保持源码位置干净
    - mkdir bld
    - cd bld
* [编译配置](https://dev.mysql.com/doc/refman/5.6/en/source-configuration-options.html)：
    - 安装基础目录：-DCMAKE_INSTALL_PREFIX=/application/mysql
    - 数据存储目录：-DMYSQL_DATADIR=/application/mysql/data
    - 建立mysql库文件libmysqld ：-DWITH_EMBEDDED_SERVER=ON
    - 添加ssl支持：-DWITH_SSL=yes
    - 默认字符集：-DDEFAULT_CHARSET=utf8
    - 默认字符序：-DDEFAULT_COLLATION=utf8_general_ci
    - 开启debug支持：-DWITH_DEBUG=1
    - 显示当前编译变量列表及对应帮助信息：-LH

```
cmake .. -DCMAKE_INSTALL_PREFIX=/application/mysql \
-DMYSQL_DATADIR=/application/mysql/data \
-DWITH_EMBEDDED_SERVER=ON \
-DWITH_SSL=yes \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DWITH_DEBUG=1 \
-LH
```

## 编译与安装
* make
* make install

## 初始化数据
* 初始化数据：/application/mysql/scripts/mysql_install_db --basedir=/application/mysql --datadir=/application/mysql/data/ --user=mysql
* 变更目录权限：chown -R mysql.mysql /application/mysql

## 变更为系统服务
- cp /application/mysql/support-files/mysql.server /etc/init.d/mysqld
- chmod +x /etc/init.d/mysqld
- sed -i '/^basedir=/s#=#&/application/mysql#' /etc/init.d/mysqld
- sed -i '/^datadir=/s#=#&/application/mysql/data#' /etc/init.d/mysqld 
- 添加为系统服务：systemctl daemon-reload
- 设置开机启动
    + systemctl enable mysqld
    + systemctl is-enabled mysqld
- 变更pid文件位置（/etc/init.d/mysqld）
    + 示例：mysqld_pid_file_path=/application/mysql/mysqld.pid
    + 用法：sed -i '/^mysqld_pid_file_path/s#=#&/application/mysql/mysqld.pid#' /etc/init.d/mysqld
    + 系统重载配置：systemctl daemon-reload
    + 正常启停控制：systemctl start mysql

## 配置环境变量
* echo "export PATH=/application/mysql/bin:\$PATH" >> /etc/profile
* source /etc/profile

# 配置文件
## 读取顺序
>mysql不会读取其他人有写权限的配置文件【o+w】
>读取顺序如下，但是后读取的文件有高优先级

* /etc/my.cnf、/etc/mysql/my.cnf
* SYSCONFDIR/my.cnf：默认是安装目录的etc子目录
* $MYSQL_HOME/my.cnf【服务端】：MYSQL_HOME为用户自定义设置，或BASEDIR、DATADIR
* defaults-extra-file： 命令行选项--defaults-extra-file
* ~/.my.cnf：用户自定义
* ~/.mylogin.cnf【客户端，较少使用】：用户登录类型配置定义

## 语法
* 配置文件和命令行语法类似【mysqld --verbose --help】，但是不包含命令行选项前导的双横向
* 井号\#或分号`;`开始的行是注释，井号也可以出现在行中间
* `[group]`：选项分组，分组名称和程序名称相同，如：
    - `[mysqld]`：mysqld程序
    - `[mysql]`：mysql程序
    - `[client]`：所有客户端程序
    - `[mysqldump]`：mysqldump程序
* opt_name：选项，与命令行--opt_name作用一样
* opt_name=value：与命令行--opt_name=value作用一样
* !include、!includedir：包含其他配置文件或目录

## 服务端配置项
>mysqld读取[mysqld] and [server]配置项，类似的mysqld_safe读取[mysqld], [server], [mysqld_safe], and [safe_mysqld] 配置项

### 运行时查看与设置
* 查看变量：SHOW VARIABLES;
* 设置变量：SET `[GLOBAL]` var_name = value;
* 统计信息和状态值查看：SHOW STATUS;

### 配置文件设置
```
# 服务端配置包含以下内容
# 命令选项：https://dev.mysql.com/doc/refman/5.6/en/server-options.html
# 系统变量：https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html
# innodb参数：https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html
# 主从复制从配置：https://dev.mysql.com/doc/refman/5.6/en/replication-options-slave.html
[client]
default-character-set = utf8               
socket = /application/mysql/mysqld.sock
# user="root"
# password="root

[mysqld]
# 杂项
lower_case_table_names = 0 #表名大小写敏感

# 连接
basedir = /application/mysql/
datadir = /application/mysql/data
socket = /application/mysql/mysqld.sock
pid_file = /application/mysql/mysqld.pid
back_log = 300 #操作系统监听队列保持的连接数【默认50 + (max_connections / 5)】
max_connections = 300 # mysql允许的连接数

# 日志
# log_err 自定义设置有bug：mysqld_safe error: log-error set to
# https://bugs.mysql.com/bug.php?id=84427
log_error = /application/mysql/mysqld.err
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /application/mysql/slow.log

# binlog设置
# expire_logs_days = 10 #保留的日志天数
log-bin = mysql-bin
log-bin-index = mysql-bin.index
binlog_format = statement # binlog格式

# 复制操作从配置
server-id = 1 
# replicate-ignore-db = mysql #忽略库mysql更新   
# replicate-ignore-table=db_name.tbl_name #忽略表db_name.tbl_name更新                            
# log-slave-updates # 默认从库不写binlog；但，从库作为其他从库的主库时，需开启binlog写入【配合log-bin设置】
# read-only=yes # 只读设置
# slave-skip-errors = 1032,1062,126,1114,1146,1048,1396  #从库更新时跳过指定错误
# relay-log-index = /usr/local/mysql/log/relaylog.index # relay-log设置
# relay-log-info-file = /usr/local/mysql/log/relaylog.info
# relay-log = /usr/local/mysql/log/relaylog
```
