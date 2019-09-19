---
title: mysql用户和权限
tags:
  - 权限
categories:
  - mysql
date: 2019-08-23 13:05:39
---

# 所有权限
* ALL：全局或者全数据库对象级别的所有权限
* ALTER：修改表结构的权限，但必须要求有create和insert权 限配合。如果是rename表名，则要求有alter和drop原表，create和 insert新表
* ALTER ROUTINE：修改或者删除存储过程、函数
* CREATE：创建存储过程、函数
* CREATE ROUTINE：创建存储过程和函数
* CREATE TABLESPACE：创建、修改、删除表空间和日志组
* CREATE TEMPORARY TABLES：创建临时表
* CREATE USER：创建、修改、删除、重命名用户
* CREATE VIEW：创建视图
* DELETE：删除行数据
* DROP：删除数据库、表、视图的权限，包括truncatetable命令
* EVENT：查询，创建，修改，删除MySQL事件
* EXECUTE：执行存储过程和函数
* FILE：在MySQL可以访问的目录进行读写磁盘文件操作，可使用 的命令包括load data infile,select ... into outfile,load file()函数
* INDEX：创建、删除索引
* INSERT：插入数据，同时在执行analyze table,optimize table,repair table语句的时候也需要insert权限
* LOCK TABLES：锁表
* PROCESS：看MySQL中的进程信息，比如执行showprocesslist,
* REFERENCES：创建外键
* RELOAD：执行flush命令，指明重新加载权限表到系统内存中， refresh命令代表关闭和重新开启日志文件并刷新所有的表
* REPLICATION SLAVE：允许slave主机通过此用户连接master以便建立主从复制关系
* REPLICATION CLIENT：执行show master status,show slave status,show binary logs命令
* SELECT：查询
* SHOW DATABASES：查看数据库
* SHOW VIEW：执行show create view命令查看视图创建的语句mysqladmin processlist, show engine等命令
* SHUTDOWN：允许关闭数据库实例，执行语句包括mysqladmin shutdown
* SUPER：执行一系列数据库管理命令，包括kill强制关闭某个连接 命令，change master to创建复制关系命令，以及create/alter/drop server等命令
* UPDATE：修改数据
* TRIGGER：允许创建，删除，执行，显示触发器

# 系统权限表
* user：存放用户账户信息以及全局级别(所有数据库)权限
    - 查询用户：select user,host mysql.user;
* db：存放数据库级别的权限
* Tables_priv：存放表级别的权限
* Columns_priv：存放列级别的权限
* Procs_priv：存放存储过程和函数级别的权限

# mysql用户
## 用户定义
* 由用户名和主机名构成，形如：'user_name'@'host_name'
* 单引号不是必须的，但有特殊字符时则必须使用
* ''@'localhost'代表匿名登录的用户
* 主机名可以是主机名或者ipv4/ipv6的地址。Localhost代表本机，127.0.0.1代表ipv4的 本机地址，::1代表ipv6的本机地址
* 允许使用%和_两个匹配字符，比如'%'代表所有主机，'%.mysql.com'代表 来自mysql.com这个域名下的所有主机，'192.168.1.%'代表所有来自192.168.1网段的主机

## 查看权限
show grants for ‘用户名’@‘ip地址’;
## 创建和授权
* 方式1：
    - 创建用户：`CREATE USER 'finley'@'localhost' IDENTIFIED BY 'some_pass';` 
    - 授权：`GRANT ALL ON *.* TO 'finley'@'localhost';`
* 方式2：`grant all on *.* to 'finley'@'localhost' IDENTIFIED BY 'some_pass';`

## 回收权限
`revoke DROP,RELOAD,SHUTDOWN,PROCESS,FILE,REPLICATION SLAVE, REPLICATION CLIENT,CREATE USER on *.* from 'finley'@'localhost';`
## 删除用户
DROP USER 'jeffrey'@'localhost';
## 权限生效
* 执行grant,revoke,set password,renameuser命令修改权限之后，MySQL会自动将修改后的权限信息同步加载到系统内存中
* 如果执行insert/update/delete操作上述的系统权限表之后，则必须再执行刷新权限命令才能同步到系统内存中，刷新权限命令包括:flush privileges/mysqladmin flush-privileges/mysqladmin reload
* 如果是修改tables和columns级别的权限，则客户端的下次操作新权限就会生效
* 如果是修改database级别的权限，则新权限在客户端执行use database命令后 生效
* 如果是修改global级别的权限，则需要重新创建连接新权限才能生效
* --skip-grant-tables可以跳过所有系统权限表而允许所有用户登录，只在特殊情况下暂时使用

## 变更密码
* 方式1：ALTER USER 'jeffrey'@'localhost' IDENTIFIED BY 'mypass';
    - 变更自身：ALTER USER USER() IDENTIFIED BY 'mypass';
* 方式2：
    - update mysql.user set authentication_string=password("新密码") where user="test" and host="localhost";
    - flush privileges;
* 方式3：SET PASSWORD FOR 'jeffrey'@'localhost' = PASSWORD('mypass');
    - 变更自身：SET PASSWORD = PASSWORD('mypass');
* 方式4(shell)：mysqladmin -u user_name -h host_name password "new_password"

## 免交互登录设置
```
# 文件~/.my.cnf设置
[client]
user='kingold'
password='zjht098_kingold'
```
