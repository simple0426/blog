---
title: mysql管理
tags:
categories:
---
# mysqladmin
服务端mysqld的管理工具，帮助信息：mysqladmin --help

# 管理命令
* system：调用系统命令
* help：帮助信息
* show `[full]`processlist;：进程列表
* show variables;：变量信息
* show `[global]` status;：状态和统计信息

# 存储过程与函数
## 查看
* 查看自定义函数：show function status;
* 查看存储过程：show procedure status;

## 定义存储过程
>批量建表的范例

```
delimiter // -- 定义分界符
drop procedure if exists create_batch_table; -- 删除已经存在的存储过程
create procedure create_batch_table() -- 创建存储过程
begin -- 开始存储过程
declare i int; -- 定义变量类型
set i = 2001; -- 设置变量
while i < 2005 do -- 循环开始
  set @create=CONCAT('create table c_fu_', i , ' like c_fu_2000;'); -- 定义创建表的语句
  select @create; -- 显示建表语句
   prepare tmt from @create; -- 预编译sql语句
   execute tmt; -- 执行sql语句
   deallocate prepare tmt;  -- 收回sql游标cursor
  set i = i + 1; -- 循环自增
end while; -- 循环结束
end // -- 结束存储过程
call create_batch_table(); -- 调用存储过程
```

## 导出存储过程与函数
>必须结合数据的导出

mysqldump crm_test -R > crm_test.sql

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
