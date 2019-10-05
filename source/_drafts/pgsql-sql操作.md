---
title: pgsql-sql操作
tags: ['psql']
categories: ['postgresql']
---
# 常用操作
## sql命令
* \h：查看sql命令帮助

## psql命令
* \?：查看psql命令帮助
* \a:切换对齐和非对齐模式
* \l：查看数据库列表
* \q：退出sql连接
* \c db：切换数据库
* \d：查看当前数据库的对象【表、视图、函数、索引】
    - \dt：查看当前路径下的表
* \d table1:查看表结构
* \du:查看用户【select * from pg_roles】

# sql操作
查询表的条目数：select count(*) from table;
## 导出导出csv
* 设置编码【保证windows下excel可读不乱码】：set client_encoding='gb18030';
* 导出数据【括号内为可执行语句】：copy (select * from xueji) to '/home/postgres/nnn.csv' with csv HEADER;
* 导入数据：copy xueji from '/home/postgres/nnn.csv' with csv HEADER;

# 用户和权限
* 修改服务监听端口（postgresql.conf)
    - listen_address=’*’
    - 允许连接的主机
* 添加连接授权（pg_hba.conf）
    - 【host  test aa   192.168.30.91/24 password】
    - 允许用户aa在主机 .91上以密码登陆方式操作test数据库
* 建立用户和授予操作权限
    - create user aa superuser password 'ceshi-aa123';
    - grant all on DATABASE test to aa;
* 更改用户密码：alter user postgres with password '123456';

## 添加只读账号
```
create role xiu LOGIN  NOSUPERUSER NOCREATEDB NOCREATEROLE  encrypted password 'wiquxiu8';
# showba库授权
grant connect on database showba to xiu;
grant usage on schema public to xiu;
grant usage on schema topology to xiu;
grant select on all tables in schema public to xiu;
grant select on all tables in schema topology to xiu;
grant select on all sequences in schema public to xiu;
grant select on all sequences in schema topology to xiu;
grant select on geography_columns to xiu;
grant select on geometry_columns to xiu;
grant select on raster_overviews to xiu;
grant select on raster_columns to xiu;
grant select on coupons_orders to xiu;
grant select on subscribe_orders to xiu;
grant select on total_orders to xiu;
# 更改新建库默认权限
ALTER DEFAULT PRIVILEGES IN SCHEMA public grant select on tables to xiu;
ALTER DEFAULT PRIVILEGES IN SCHEMA public grant select on sequences to xiu;
ALTER DEFAULT PRIVILEGES IN SCHEMA topology grant select on tables to xiu;
ALTER DEFAULT PRIVILEGES IN SCHEMA topology grant select on sequences to xiu; 
```

# 备份和恢复
## pg_dump命令
* -F：指定输出备份格式
    - 默认为明文格式，只能使用psql命令恢复
    - custom格式使用pg_restore命令恢复
* -a：只导出库数据
* -s：只导出数据库结构
* -t：备份指定表
* -T：不备份特定表
* -C：包含创建数据库命令
* -f：指定备份输出文件或目录
* -d：需要备份的数据库

## 备份范例
* 备份数据库结构 pg_dump –Fc –s –f test.sql test
* 备份数据库 pg_dump –Fc –f test.all test
* 备份表数据 pg_dump –Fc –a –t xuji –f xuji.db test
* 备份表    pg_dump –Fc –t xuji –f xuji.all test

## pg_restore命令
* -a：只恢复库数据
* -s：只恢复数据库结构
* -t：只恢复指定表
* -d：连接的数据库

## 恢复范例
* 恢复数据库结构 pg_restore –s –d test test.sql
* 恢复数据库数据 pg_restore –a –d test test.db
* 恢复数据库 pg_restore  –d test test.all
* 恢复表结构 pg_restore –s –t xuji –d test xuji.sql
* 恢复表数据 pg_restore –a –t xuji –d test xuji.db
* 恢复表  pg_restore –t xuji –d test xuji.all

## 明文导出的备份和恢复
* 备份：pg_dump -h 10.150.10.210 -p 5432 -U postgres showba > showba
* 恢复：psql -h 10.150.10.40 -p 5432 -U postgres showba < /home/db/db

# 系统管理
* 查看当前会话连接：elect * from pg_stat_activity;
* 终止一个会话：pg_terminate_backend(pidint)
