---
title: mysql管理
tags:
  - 存储过程
  - mysqladmin
  - 锁表
categories: ['mysql']
---
# mysqladmin
服务端mysqld的管理工具，帮助信息：mysqladmin --help

# 管理命令
* system：调用系统命令
* help：帮助信息
* show `[full]`processlist;：进程列表
* show variables;：变量信息
* show `[func_name]` status;：状态和统计信息

# 使用优化
* 能用定长char的就不用varchar
* 查询时，尽量不要使用select \*，同时要用where条件匹配
* 尽量使用批量插入

# 锁表处理
## 情景1
* 现象与解释：mysql命令行执行时，sql挂起；使用show processlist查看，该sql线程状态为【waiting for table metadata lock】；这是由于该表有正在执行的其他长事务（sql），进而阻塞了同表的后续操作。
* 处理
    - 使用【show processlist】查看长事务的线程id
    - 使用kill命令杀死改线程

## 情景2
* 现象和解释：对该表进行操作时，客户端显示锁表（比如：java客户端中出现错误【Lock wait timeout exceeded; try restarting transaction】，但是通过show processlist命令看不到该表的任何操作；实际上该表存在未提交的事务，可以在
information_schema.innodb_trx中查看到。
* 处理
    - 使用【select * from information_schema.innodb_trx\G】查找相关表的线程id
    - 使用kill命令杀死该线程【由于是事务类型操作，杀死线程后，事务会回滚】
  