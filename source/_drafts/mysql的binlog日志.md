---
title: mysql的binlog日志
tags: ['binlog']
categories: ['mysql']
---
# 参考
* [MySQL 通过 binlog 恢复数据](https://learnku.com/articles/20628)

# 日志分类
* 错误日志：mysql运行过程中的错误信息
    - 功能开关：无
    - 文件位置：log_error
* 一般日志：记录 mysql 正在运行的语句，包括查询、修改、更新等的每条 sql
    - 功能开关：general_log=OFF
    - 文件位置：general_log_file
* 慢查询日志：记录查询比较耗时的 SQL 语句
    - 功能开关：slow_query_log=OFF
    - 阈值设置：long_query_time
    - 文件位置：slow_query_log_file
* binlog日志：记录数据修改记录，包括创建表、数据更新等
    - 功能开关：log_bin=ON
    - 文件位置：log_bin_index、log_bin_basename

# binlog功能
* 数据恢复：当数据库误删或者发生不可描述的事情时，可以通过 binlog 恢复到某个时间点的数据。
* 主从复制：当有数据库更新之后，主库通过 binlog 记录并通知从库进行更新，从而保证主从数据库数据一致；

# binlog文件切换
* 文件大小达到max_binlog_size参数大小
* 执行 flush logs 命令
* 重启 mysql 服务

# binlog日志格式
通过binlog_format设置binlog格式，可选参数值如下：

* statement：记录数据库执行的原始 SQL 语句
* row：记录具体的行的修改，这个为目前默认值
* mixed ：因为上边两种格式各有优缺点，所以就出现了 mixed 格式

# binlog日志查看
因为 binlog 是二进制文件，不能像其他文件一样，直接打开查看。但 mysql 提供了 binlog 查看工具 mysqlbinlog，可以解析二进制文件。  
不同格式的binlog查看方式也有不同，具体如下：  

* statement：执行 mysqlbinlog /path/bin-log.000001，可以直接看到原始执行的 SQL 语句
* row：则可读性没有那么好，但仍可通过参数使文档更加可读 mysqlbinlog -v /path/bin-log.000001

# binlog日志删除
* 设置日志过期变量：
    - 查看：show variables like 'expire_logs_days';
    - 设置：set global expire_logs_days = 3;
* 手动删除日志
    - 删除3天前：PURGE MASTER LOGS BEFORE DATE_SUB( NOW( ), INTERVAL 3 DAY);
    - 清除日志文件：PURGE MASTER LOGS TO 'MySQL-bin.010';
    - 删除指定日期前：PURGE MASTER LOGS BEFORE '2008-06-22 13:00:00';

# mysqlbinlog其他参数
* --base64-output=DECODE-ROWS：将base64编码的内容解码，以便于查看内容
* -start-datetime --stop-datetime 解析某一个时间段内的 binlog
* -start-position --stop-position 解析在两个 position 之间的 binlog
