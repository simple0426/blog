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

# 系统管理
* 会话管理
    - 查看当前会话连接：select * from pg_stat_activity;
    - 取消一个查询：pg_cancel_backend(pid int)
    - 终止一个会话：pg_terminate_backend(pidint)
* 变量
    - 查询：select current_setting(setting_name)或show var
    - 设置：select set_config(setting_name, new_value, is_local)或【SET var TO val;】
        + 如果 is_local 设置为 true，那么新数值将只应用于当前事务。 如果你希望新的数值应用于当前会话，那么应该使用 false。
* 数据库对象存储位置
    - pg_relation_filenode(relation regclass)：获取指定对象的文件节点编号(通常为对象的oid值)。
    - pg_relation_filepath(relation regclass)：获取指定对象的完整路径名。

```
showba=# select pg_relation_filenode('auth_group');
pg_relation_filenode
----------------------
                16388
select pg_relation_filepath('auth_group');
pg_relation_filepath
----------------------
base/16387/16388
```

* 数据库对象尺寸大小
    - pg_column_size(any)
    - pg_tablespace_size(oid/name)
    - pg_relation_size(oid/name)
    - pg_database_size(oid/name)
    - pg_size_pretty(bigint):把字节计算的尺寸转换成一个人类易读的尺寸单位
        + select pg_size_pretty(pg_database_size('test'));
* 备份管理
    - 设置开始执行备份：pg_start_backup(label_text text)：pg_start_backup 接受一个参数，这个参数可以是任意用户为备份定义的标签。 （通常这是备份转储文件存放所在的名字。）这个函数向数据库集群的数据目录写入一个备份标签文件， 然后以文本方式返回备份的起始 WAL 偏移。（用户不需要注意这个结果值，提供他只为了万一需要的场合。）
    - 完成执行的在线备份：pg_stop_backup()：pg_stop_backup 删除 pg_start_backup 创建的标签文件， 并且在 WAL 归档区里创建一个备份历史文件。这个历史文件包含给予 pg_start_backup 的标签， 备份的起始与终止 WAL 偏移量，以及备份的起始和终止时间。返回值是备份的终止 WAL 偏移 （同样也可能没有什么用）。

# 表空间
* 功能：可以指定数据库对象的存储位置，方便扩展存储空间
* 创建：
    - 语法：CREATE TABLESPACE tb_space LOCATION '/home/postgres/db0/tb_space';
    - 说明：os_path必须是空的、postgresql帐号有权的目录。创建表空间的用户必须是superuser，创建完表空间之后，可以将表空间的create权限赋给普通用户使用！
* 使用：
    - 可使用对象：表、index、数据库：在创建这些对象时，可以显式的指定tablespace tals_name子句指示对象使用的表空间。
    - 说明：
        + postgresql允许通过符号链接简化表空间的实施，那在不支持符号链接的os上就无法简化，只能显式的创建所需的表空间了！
        + 表空间是和单个数据库无关的，他被所有的数据库使用。因为，表空间只有没有任何对象使用时，才能drop掉
    - 语法：create database test tablespace tb_space;
