---
title: mysql介绍
tags:
  - InnoDB
  - 存储引擎
  - 字符集
categories:
  - mysql
date: 2019-09-19 18:06:50
---

# 版本
* mysql-5.5
* mysql-5.6【牧客，测试环境版本：5.6.43】
* mysql-5.7【中冀汇通，全环境版本：5.7.17】
* mariadb
* mysql-8.0

# 下载地址
* mysql相关资源下载目录：https://www.mysql.com/products/community/
* 社区版server下载地址：https://dev.mysql.com/downloads/mysql/

# 体系架构
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/mysql-arch.jpg)

* 连接池组件
* 管理服务和工具组件
* SQL接口、分析器、优化器、缓冲组件
* 存储引擎
* 物理文件

# 存储引擎
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/mysql-engine.jpg)
## 查看
* 查看支持的存储引擎：show engines;【常见引擎如上】
* 查看当前的存储引擎选项：show variables like '%engine%';
* 查看表的存储引擎：show create table tb_name;

## 设置与修改
* alter table table_name engine=innodb；
* 将表结构导出使用sed替换，将数据导出，再将表结构导入，数据导入
* 使用mysql_convert_table_format命令
* 建表时使用engine=innodb参数

## InnoDB与MyISAM对比
|            MyISAM            |                InnoDB                |
|------------------------------|--------------------------------------|
| 不支持事务                   | 支持事务、分区、表空间               |
| 写操作时表级别锁定           | 写操作时行级别锁定                   |
| 只缓存索引                   | 缓存索引和数据                       |
| 读操作会阻塞写操作           | 读写操作相对独立【根据事务隔离级别】 |
| 支持全表索引，不支持外键约束 | 支持外键约束，不支持全表索引         |

## MyISAM
* 物理文件：每一个myisam表对应于硬盘上的3个文件。这三个文件有一样的文件名，但是有不同的扩展名以指示其用途：
    - frm：保存表的定义，但这个文件不是myisam引擎的一部分，而是mysql数据库本身的一部分
    - MYD(data)：保存表的数据
    - MYI(index)：表的索引文件
* 应用：
    - 并发较低的业务
    - 以读为主的业务
    - 不需要事务支持的业务
* 优化：
    - 尽量索引及使用缓存
    - 尽量顺序操作将数据插入尾部
    - 分解复杂操作，延迟写入，批量插入
    - 降低并发数，高并发应用引入排队机制

## InnoDB
* 物理文件：
    - ibdata1，默认表空间文件，通过innodb_data_file_path参数设置，所有基于innodb存储引擎的表都会存储在该文件中
        + 如果想基于每个表单独生成一个表空间文件，可以设置参数innodb_file_per_table为ON，这样表的数据、索引、插入缓冲等信息会存储在单独的表空间文件中，但是其余的信息还是存储在默认的表空间文件中
    - frm：表结构定义文件，mysql里每个表和视图都对应一个frm文件。frm文件时mysql数据库的一部分，和存储引擎无关
    - ibd：单独的表空间文件
    - ib_logfile：innodb存储引擎的事务日志；在数据库实例意外宕机，重启后可以利用重做日志恢复到宕机前的一致性状态
* 应用：
    - 支持高并发，但需要确保查询时通过索引完成
    - 支持事务
    - 数据更新频繁
* 优化：
    - 避免全表扫描，因为会使用表锁
    - 尽可能缓存所有索引和数据，提高响应速度，降低磁盘io
    - 在大批量小插入时，尽量自己控制事务而不要使用autocommit自动提交
    - 合理设置innodb_flush_log_at_trx_commit参数值，不要过度追求安全性
    - 避免主键更新，因为这会带来大量的数据移动
    - 主键尽可能小，避免给secondary index带来过大的空间负担

## 事务
### 简介
* 在mysql中只有使用了innodb引擎的数据库才支持事务
* 事务处理可以用来维护数据库的一致性，保证多条sql语句要么全部执行，要么全部不执行
* 事务用来管理insert、update、delete语句

### 特点-ACID
* 原子性（Atomicity）：一个事务（transaction）中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。
* 一致性（Consistency）：在事务开始之前和事务结束以后，数据库的完整性没有被破坏。
* 隔离性（Isolation）：数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。
* 持久性（Durability）：事务处理结束后，对数据的修改是永久性的

### 隔离级别
- 查询：SELECT @@tx_isolation;【默认REPEATABLE-READ】
- 设置：set session transaction isolation level 隔离级别

|     隔离级别     |       描述       | 解决的问题 |                          引发的新问题                          |
|------------------|------------------|------------|----------------------------------------------------------------|
| serializable     | 串行读取数据     | 虚读       | 数据库就变成了单线程访问的数据库，导致性能降低很多             |
| repeatable read  | 可重复读         | 不可重复读 | 虚读：前后读取到表中的记录数不一样，读取到了其他事务插入的数据 |
| read committed   | 读取提交的记录   | 脏读       | 不可重复读：一个事务读取表中的某一行数据时，多次读取结果不一致 |
| read uncommitted | 读取未提交的数据 | -          | 脏读：一个事务读取了另外一个事务未提交的数据                   |

### 开启事务
mysql命令行下默认设置下，事务都是自动提交的；开启事务有以下两种方式：

* 关闭事务自动提交模式：SET AUTOCOMMIT=0
* 自动提交模式下，使用命令显式开启一个事务：begin或start transaction

### 事务控制语句
* begin或start transaction：开启一个事务
* commit：提交一个事务
* rollback：结束用户的事务，并撤销之前的修改
* savepoint identifier：在事务中创建一个保存点
* release savepoint identifier：删除一个事务保存点
* rollback to identifier：把事务回滚到保存点

# 字符集
## 概念
* 字符集（character set）：字符和编码对组成的集合
* 字符序（collection）：同一个字符集内字符之间的比较规则

## 字符集变量
### 字符集相关变量
* 查看支持的字符集：SHOW CHARACTER SET;
* 查看当前的字符集设置：show variables like '%character%';
    - character_set_client ：客户端使用的字符集，相当于网页中那个的字符集设置
    - character_set_connection：客户端连接数据库的使用的字符集
    - character_set_results：数据库给客户端返回时使用的字符集设定
    - ~~character_set_system~~：数据库所在操作系统使用的字符集，character_set_system是个只读数据不能更改
    - character_set_server：数据库实例启动时的默认字符集，也是建库时的默认字符集
    - ~~character_set_database~~：当前数据库的字符集设置，可通过建库语句查看。

### 字符序相关变量
* 查看支持的字符序：show collation;
* 查看当前的字符序设置：show variables like '%collation%';
    - collation_server：数据库实例默认校对规则
    - collation_database ：当前数据库的字符序设置
    - collation_connection：连接字符集校对规则

### 字符集的转换过程
* 客户端连接数据库时，如果不指定字符集【default-character-set】，则使用服务端默认设置【character-set-server】
* mysql server收到请求时将请求从character_set_client转换为character_set_connection
* 进行内部操作前将请求数据从character_set_connection转换为内部操作字符集，确定顺序如下：
    - 使用字段的字符集
    - 使用表的字符集
    - 使用库的字符集【建库时，未指定则使用character-set-server】
* 将操作结果从内部操作字符集转换为character_set_results返回给客户端

## 客户端设置
* 设置参数：default-character-set【服务端已不支持此参数】
* 影响变量：
    - character_set_client 
    - character_set_connection
    - character_set_results
    - collation_connection

## 服务端设置
### 字符集的继承关系
* 创建数据库没有指定字符集，使用character_set_server设定值
* 创建表没有指定字符集，默认使用数据库的的字符集
* 字段没有设置字符集，默认使用表的字符集

### server级别设置
* 设置参数：character-set-server、collation-server
* 影响变量：
    - character_set_server
    - character_set_database
    - collation_server
    - collation_database
* 设置方式
    - 服务启动时命令行指定：mysqld --character-set-server=latin1
    - 配置文件my.cnf指定：character-set-server = utf8
    - 运行时修改：SET character_set_server = utf8 ;

### database级别设置
* 查看库字符集：SHOW CREATE DATABASE db_name;
* 建库时设置：`CREATE DATABASE db_name [[DEFAULT] CHARACTER SET charset_name] [[DEFAULT] COLLATE collation_name]`
* 修改数据库的默认字符集：`ALTER DATABASE db_name DEFAULT CHARACTER SET character_name [COLLATE ...];`
    - 范例：alter database shiyan default character set utf8 COLLATE utf8_general_ci;

### table级别设置
* 查看表字符集：SHOW CREATE TABLE tbl_name;
* 建表时设置：CREATE TABLE tb_name ( id INT NOT NULL) DEFAULT CHARACTER SET = utf8;
* 修改表默认字符集：`ALTER TABLE tbl_name DEFAULT CHARACTER SET character_name [COLLATE...];`
    - 范例：ALTER TABLE logtest DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
* 修改表的当前字符集：`ALTER TABLE tbl_name CONVERT TO CHARACTER SET character_name [COLLATE ...]`
    - 范例：ALTER TABLE logtest CONVERT TO CHARACTER SET utf8 COLLATE utf8_general_ci; 

### column级别设置
* 查看字符字符集：SHOW FULL COLUMNS FROM tbl_name;
* 新增字段设置：ALTER TABLE test_table ADD COLUMN char_column VARCHAR(25) CHARACTER SET utf8;
* 修改当前字段：`alter table table_name modify column column_attr character set character_name [COLLATE ...]`
    - 范例：alter table test1 modify name char(10) character set utf8 COLLATE utf8_general_ci; 

# 日志分类
* 一般日志：记录 mysql 正在运行的语句，包括查询、修改、更新等的每条 sql
    - 功能开关：general_log=OFF
    - 文件位置：general_log_file
* 错误日志：mysql运行过程中的错误信息
    - 功能开关：无
    - 文件位置：log_error
* 慢查询日志：记录查询比较耗时的 SQL 语句
    - 功能开关：slow_query_log=OFF
    - 阈值设置：long_query_time
    - 文件位置：slow_query_log_file
* binlog日志：记录数据修改记录，包括创建表、数据更新等
    - 功能开关：log_bin=ON
    - 文件位置：log_bin_index、log_bin_basename

# SQL操作分类
* DDL(数据定义语言)：create、alter、drop
* DML(数据修改语言)：insert、delete、update
* DQL(数据查询语言)：select、order by、group by、having
* DCL(数据控制语言)：create user、grant、revoke、drop user

---
# 参考
* [字符集设置](https://www.cnblogs.com/chyingp/p/mysql-character-set-collation.html)
