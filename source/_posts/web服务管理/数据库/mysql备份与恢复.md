---
title: mysql备份与恢复
tags:
  - mysqlbinlog
  - mysqldump
categories:
  - mysql
date: 2019-09-25 17:52:05
---

# 备份命令mysqldump
* -u 用户名
* -p 密码
* -h 主机
* -P 端口
* -S 指定链接mysql的socket
* --default-character-set  设置导出的字符集
* -F, --flush-logs  在每个数据库备份前，刷新binlog日志
* --compact 去除注释部分
* --single-transaction      适用于innodb，保持数据一致性，隔离级别为：repeatable-read
* -A, --all-databases 备份所有库，等同于--databases DB1 DB2 。。。
* -B, --databases  导出多个数据库
* --ignore-table=database.table 备份库时忽略指定的表【可多次使用，每次一个表】
* -d, --no-data 导出表结构【不导出数据】
* -t, --no-create-info 导出数据【不添加create table语句】
* -R, --routines  导出存储过程和自定义函数
* -E, --events 导出事件
* --skip-triggers 不导出触发器【默认导出--triggers】

# 备份库表
## 备份所有库
`mysqldump [OPTIONS] --all-databases [OPTIONS]`
## 备份多个库
`mysqldump [OPTIONS] --databases [OPTIONS] DB1 [DB2 DB3...]`
## 备份单个库
`mysqldump [OPTIONS] database [tables]`

* 第1个字符串为数据库
* 第2个字符串及之后字符串为数据表

## 备份表
mysqldump db1 table1 > db1_table1.sql
## 备份表数据
mysqldump db1 table1 -t > db1_table1.sql
## 备份表结构
mysqldump db1 table1 -d > db1_table1.sql
## 分库分表备份
```
DB_NAME=`mysql -uadmin -pzjht0724# -e "show databases;"|sed '1,2d'`
mkdir  /home/zj-db/backup/$(date +%F) -p
cd /home/zj-db/backup/$(date +%F)
# backup every database with every table
for db in $DB_NAME
do
    if [[ "$db" != 'information_schema' && "$db" != 'performance_schema' && "$db" != 'mysql' && "$db" != 'sys' ]];then
        mkdir /home/zj-db/backup/$(date +%F)/$db -p
            TB_NAME=`mysql -uadmin -pzjht0724# -e "show tables from $db;"|sed '1d'`
            for tb in $TB_NAME
            do
                    mysqldump -uadmin -pzjht0724# --lock-tables=0 $db $tb > /home/zj-db/backup/$(date +%F)/${db}/${tb}.sql
            done
        # 备份压缩及加密
        zip -r -Pzjht1234 ${db}.zip $db
        rm -rf $db
    fi
done
```
# 备份选项
## 全量备份与增量备份
* 全量备份：使用mysqldump命令导出的数据
* 增量备份：由于mysql的数据修改操作都记录在binlog中，执行日志文件切割轮转后，新增的binlog文件即视为增量备份
    - 可以使用mysqldump -F或mysqladmin flush-logs切割轮转binlog文件

## 物理备份和逻辑备份
|          逻辑备份          |              物理备份              |
|----------------------------|------------------------------------|
| 导出sql语句                | 直接复制数据文件                   |
| mysqldump                  | scp、cp、rsync等                   |
| 速度较慢                   | 速度较快                           |
| 支持server跨版本备份与恢复       | 要求导入和导出的server版本尽量一致 |
| 数据量小于100G使用逻辑备份 | 数据量大于100G使用物理备份         |

## 备份策略
* 每天凌晨做一个全量备份
* 在全量备份的同时执行日志轮转切割【flush-logs】作为增量备份点

# binlog介绍
## binlog介绍
* 数据恢复：当数据库误删或者发生不可描述的事情时，可以通过 binlog 恢复到某个时间点的数据。
* 主从复制：当有数据库更新之后，主库通过 binlog 记录并通知从库进行更新，从而保证主从数据库数据一致；

## binlog功能设置
* 功能开关及文件名设置：log_bin=mysql-bin
- binlog日志格式：binlog_format=statement，可选参数值如下
    + statement：记录数据库执行的原始 SQL 语句
    + row：记录具体的行的修改，这个为目前默认值
    + mixed ：因为上边两种格式各有优缺点，所以就出现了 mixed 格式
* binlog文件切换条件
    - 文件大小达到max_binlog_size参数大小
    - 执行 flush logs 命令
    - 重启 mysql 服务

## binlog删除
* 设置日志过期变量：
    - 查看：show variables like 'expire_logs_days';
    - 设置：set global expire_logs_days = 3;
* 手动删除日志
    - 删除全部：reset master;
    - 清除MySQL-bin.010之前的日志文件：PURGE MASTER LOGS TO 'MySQL-bin.010';
    - 删除3天前：PURGE MASTER LOGS BEFORE DATE_SUB( NOW( ), INTERVAL 3 DAY);
    - 删除指定日期前：PURGE MASTER LOGS BEFORE '2008-06-22 13:00:00';

## binlog查看
因为 binlog 是二进制文件，不能像其他文件一样，直接打开查看。但 mysql 提供了 binlog 查看工具 mysqlbinlog，可以解析二进制文件。  
不同格式的binlog查看方式也有不同，具体如下：  

* statement：执行 mysqlbinlog /path/bin-log.000001，可以直接看到原始执行的 SQL 语句
* row：则可读性没有那么好，但仍可通过参数使文档更加可读 mysqlbinlog -v /path/bin-log.000001

# binlog解析命令mysqlbinlog
* -u, --user=name
* -h, --host=name 
* -p, --password[=name]
* -P, --port=#
* -S, --socket=name
* -r, --result-file=name：导出内容到文件【重定向输出】
* --server-id=#：只导出指定server-id的内容
* --set-charset=name：设置字符集
* --base64-output=DECODE-ROWS：将base64编码的内容解码，以便于查看内容
* -d, --database=name 只导出指定数据库的内容
* -o, --offset=# 跳过开始的n行内容
* -start-datetime、--stop-datetime 解析两个时间点之间的 binlog，时间类型为DATETIME和TIMESTAMP
    - 范例：mysqlbinlog mysql-bin.000001 -d seo -vv --base64-output=DECODE-ROWS --start-datetime='2019-06-01 00:00:00'  --stop-datetime='2019-07-01 00:00:00' -r seo2.sql
* -start-position、--stop-position 解析两个position 之间的 binlog【包含开始点，不包含结束点】
    - 范例：mysqlbinlog mysql-bin.000001 -d seo -vv --base64-output=DECODE-ROWS --start-position=46301253  --stop-position=46328958 -r seo3.sql

# binlog恢复数据
## 指定库恢复
将全量备份之后的binlog使用-d参数后导出指定数据库
## 指定表恢复
* 在测试库上将指定表所在库的数据恢复
* 将指定表的数据导出
* 将导出的表数据恢复到指定的库

# 备份与恢复操作
>开启binlog，并且binlog不能有过期删除操作

* 【备份】mysqldump备份数据【全量备份】，添加-F参数切换binlog文件，并获取备份后的第一个binlog文件名【增量备份】
* 【问题】服务出现异常，查明是因为mysql出现问题操作
* 【停止服务】停止mysql的对外服务【可通过iptables封禁3306端口】
* 【保护现场】备份全量备份数据和所有的binlog文件，防止数据二次破坏
* 【获取增量备份（删除问题sql）】找到全量备份后的所有binlog，并使用mysqlbinlog工具将binlog转换为sql，同时删除有问题的sql语句
* 【恢复全量备份】
* 【恢复增量备份】
* 【开启服务】检查恢复结果，确认正常后，开启对外服务

# 参考
* [MySQL 通过 binlog 恢复数据](https://learnku.com/articles/20628)
