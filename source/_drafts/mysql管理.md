---
title: mysql管理
tags: ['mysql']
categories: ['DataBase']
---
# 管理
* 查看表结构：desc tablename;
* 查看建表语句：show create table tablename;

# 字符集设置
## 查看支持的字符集
show charset;
## 设置全局默认字符集
* 设置：character-set-server = utf8mb4【my.cnf】
* 查看：show variables like '%character%';

## [修改已有库表字符集][mysql-character]
* 修改数据库默认字符集：ALTER DATABASE db_name DEFAULT CHARACTER SET character_name [COLLATE ...];
    - 范例：alter database shiyan default character set utf8 COLLATE utf8_general_ci;
    - 查看库字符集：SHOW CREATE DATABASE db_name; 
* 修改表默认字符集：ALTER TABLE tbl_name DEFAULT CHARACTER SET character_name [COLLATE...]; 
    - 范例：ALTER TABLE logtest DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
    - 查看表字符集：SHOW CREATE TABLE tbl_name;
* 修改当前表的字符集：ALTER TABLE tbl_name CONVERT TO CHARACTER SET character_name [COLLATE ...]
    - 范例：ALTER TABLE logtest CONVERT TO CHARACTER SET utf8 COLLATE utf8_general_ci; 
* 修改当前字段字符集：alter table table_name modify column column_attr character set character_name [COLLATE ...]
    - 范例：alter table test1 modify name char(10) character set utf8 COLLATE utf8_general_ci; 
    - 查看字符字符集：SHOW FULL COLUMNS FROM tbl_name;

# binlog处理
* 设置日志过期变量：
    - 查看：show variables like 'expire_logs_days';
    - 设置：set global expire_logs_days = 3;
* 手动删除日志
    - 删除3天前：PURGE MASTER LOGS BEFORE DATE_SUB( NOW( ), INTERVAL 3 DAY);
    - 清除日志文件：PURGE MASTER LOGS TO 'MySQL-bin.010';
    - 删除指定日期前：PURGE MASTER LOGS BEFORE '2008-06-22 13:00:00';

[mysql-character]: https://blog.csdn.net/weixin_40539892/article/details/80564842


