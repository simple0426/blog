---
title: mysql存储过程
tags:
categories:
---
# 批量建表
## 存储过程方式
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
## shell脚本方式
```
tb_list="c_fu c_fu_file e_email e_email_body e_email_file e_email_to e_email_track f_mem_upload"
for index in `seq 2001 2999`;do
  for tb in $tb_list;do
    mysql -h 127.0.0.1 -uuser -p'pass_str' crm_big3 -e "create table ${tb}_${index} like ${tb}_2000"
  done
  echo -e "$index is created"
  sleep 0.1
done
```
## python脚本方式
```
import pymysql

tb_list = ['c_fu', 'c_fu_file', 'e_email', 'e_email_body', 'e_email_file', 'e_email_to', 'e_email_track', 'f_mem_upload']
conn = pymysql.connect(host='127.0.0.1', user='user', password='pass_str', database='crm_big3', charset='utf8')
for index in range(2001, 2003):
    for table in tb_list:
        cursor = conn.cursor(cursor=pymysql.cursors.Cursor)
        cursor.execute('create table %s_%s like %s_2000' % (table, index, table))
        cursor.close()
conn.close()
```
## 方式对比
* 存储过程，每次新建一类表，在命令使用`mysql crm_big3 < 1.sql`方式执行，可以达到一次连接建一类表
* shell方式，每个连接只能建一个表，所以会占用数据库连接
* python方式，一个连接可以把表全部建立
