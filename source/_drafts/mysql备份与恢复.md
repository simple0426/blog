---
title: mysql备份与恢复
tags: ['mysqldump']
categories:
    - mysql
---
# 命令参数
* -u 用户名
* -p 密码
* -h 主机
* -S 指定链接mysql的socket
* -A 备份所有库
* -B 【--databases】导出数据库列表，单个库时可省略
* --tables 导出的表列表（单个表时可省略）
* -d 【--no-data】 导出表结构【不导出数据】
* -t 【--no-create-info】 导出数据【不添加create table语句】
* -n 【--no-create-db】 导出数据【不添加create database语句】
* -R 【--routines】 导出存储过程和自定义函数
* -E 【--events】 导出事件
* --triggers 默认导出触发器【--skip-triggers屏蔽导出】
* -F 刷新binlog日志
* --compact 去除注释部分

# 使用范例
## 导出库
mysqldump db1 > db1.sql
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
