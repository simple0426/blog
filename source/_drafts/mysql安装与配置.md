---
title: mysql安装与配置
tags:
categories:
---
# 源码安装
## [依赖安装](https://dev.mysql.com/doc/refman/5.6/en/source-installation-prerequisites.html)
* cmake
* make OR gmake
* GCC 4.2.1 or later
    - centos系列使用gcc*
* SSL library
* ncurses
    - ubuntu16：libncurses5-dev、libncurses5

## [预配置](https://dev.mysql.com/doc/refman/5.6/en/installing-source-distribution.html)
* groupadd mysql
* useradd -r -g mysql -s /bin/false mysql
* mkdir /application

## 软件解压
tar zxvf mysql-VERSION.tar.gz
## 编译配置
* cd mysql-VERSION
* 保持源码位置干净
    - mkdir bld
    - cd bld
* [编译配置](https://dev.mysql.com/doc/refman/5.6/en/source-configuration-options.html)：
    - 安装基础目录：-DCMAKE_INSTALL_PREFIX=/application/mysql
    - 数据存储目录：-DMYSQL_DATADIR=/application/mysql/data
    - 建立mysql库文件libmysqld ：-DWITH_EMBEDDED_SERVER=ON
    - 添加ssl支持：-DWITH_SSL=yes
    - 默认字符集：-DDEFAULT_CHARSET=utf8
    - 默认字符序：-DDEFAULT_COLLATION=utf8_general_ci
    - 开启debug支持：-DWITH_DEBUG=1
    - 显示当前编译变量列表及对应帮助信息：-LH

```
cmake .. -DCMAKE_INSTALL_PREFIX=/application/mysql \
-DMYSQL_DATADIR=/application/mysql/data \
-DWITH_EMBEDDED_SERVER=ON \
-DWITH_SSL=yes \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DWITH_DEBUG=1 \
-LH
```

## 编译与安装
* make
* make install

## 初始化数据
* 初始化数据：/application/mysql/scripts/mysql_install_db --basedir=/application/mysql --datadir=/application/mysql/data/ --user=mysql
* 变更目录权限：chown -R mysql.mysql /application/mysql

## 变更为系统服务
- cp support-files/mysql.server /etc/init.d/mysqld
- chmod 700 /etc/init.d/mysqld
- sed -i '/^basedir=/s#=#&/application/mysql#' /etc/init.d/mysqld
- sed -i '/^datadir=/s#=#&/application/mysql/data#' /etc/init.d/mysqld 
- 开机启动
    + sysv-rc-conf mysqld on【ubuntu】
    + systemctl enable mysqld【centos】

## 默认配置文件
cp support-files/my-default.cnf /etc/my.cnf

## 配置环境变量
* echo "export PATH=/application/mysql/bin:\$PATH" >> /etc/profile
* source /etc/profile
