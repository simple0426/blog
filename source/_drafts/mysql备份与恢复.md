---
title: mysql备份与恢复
tags: ['mysqldump']
categories:
    - mysql
---
# 使用方法
* 备份表：

* 

# 命令参数
* -u 用户名
* -p 密码
* -h 主机
* -P 端口
* -S 指定链接mysql的socket
* --default-character-set  设置导出的字符集
* -F, --flush-logs  刷新binlog日志
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

# 使用范例
## 备份所有库
`mysqldump [OPTIONS] --all-databases [OPTIONS]`

## 备份多个库
`mysqldump [OPTIONS] --databases [OPTIONS] DB1 [DB2 DB3...]`

## 导出单个库
mysqldump db1|gzip > db1.sql.gz## 备份表
`mysqldump [OPTIONS] database [tables]`

* 第1个字符串为数据库
* 第2个字符串及之后字符串为数据表

## 导出表
mysqldump db1 table1 > db1_table1.sql
## 导出表数据
mysqldump db1 table1 -t > db1_table1.sql
## 导出表结构
mysqldump db1 table1 -d > db1_table1.sql

# 存储过程与函数
>必须结合数据的导出
## 查看
* 查看自定义函数：show function status;
* 查看存储过程：show procedure status;

## 导出存储过程与函数
mysqldump crm_test -R > crm_test.sql
