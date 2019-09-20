---
title: mysql故障处理
tags:
categories:
---
# 锁表
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
