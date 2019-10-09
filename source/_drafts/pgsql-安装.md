---
title: pgsql-安装
tags: postgresql
categories: ['postgresql']
---
# 依赖安装
>示例ubuntu16.04下的软件包

* make
* gcc
* readline：libreadline6 libreadline6-dev【用于sql命令行提示】
* zlib：zlib1g zlib1g-dev【备份和恢复时压缩】
* libpython2.7 libpython2.7-dev libpython3.5 libpython3.5-dev【python语言支持】
* ssl：OpenSSL【连接加密】
* xml2：libxml2 libxml2-dev
* xslt：libxslt1.1  libxslt1-dev

# 安装
>软件版本：v9.6.15

* ./configure --with-python --with-openssl --with-libxml --with-libxslt
* make 
* sudo make install

# 链接库设置
* 实时设置链接库【官方】
    - 【普通用户】LD_LIBRARY_PATH=/usr/local/pgsql/lib;export LD_LIBRARY_PATH
    - 【root】/sbin/ldconfig /usr/local/pgsql/lib
* 静态化设置：echo /usr/local/pgsql/lib >> /etc/ld.so.conf;ldconfig

# 环境变量设置
```
echo "PATH=/usr/local/pgsql/bin:\$PATH" >> /etc/profile
echo "export PATH" >> /etc/profile
source /etc/profile
```

# 初始化数据库
```
adduser postgres
mkdir /usr/local/pgsql/data
chown postgres /usr/local/pgsql/data
su - postgres
initdb -D /usr/local/pgsql/data --no-locale
pg_ctl -D /usr/local/pgsql/data -l logfile start
createdb test
psql test
```

# 常用控制命令
* pg_dump：备份
* pg_restore：还原
* psql：数据库连接
* vacuumdb –a (–d database) -z：整理数据库
* pg_ctl -D /usr/local/pgsql/data -l logfile start：数据库启停

# postgis安装
>postgis:PostgreSQL的空间和地理对象
## [依赖安装](http://www.postgis.net/docs/postgis_installation.html#install_requirements)
* PostgreSQL9.4以上
* gcc
* make
* proj4：proj-bin
* geos：libgeos-c1v5 libgeos-dev 
* xml2：libxml2 libxml2-dev
* json-c：libjson-c2 libjson-c-dev
* gdal：libgdal20 gdal-bin libgdal-dev 
* llvm：llvm-6.0 llvm-6.0-dev 

## 安装
* ./configure --with-pgconfig=/usr/local/pgsql/bin/pg_config
* make
* sudo make install

## 开启postgis
* postgis：psql -d test -c "CREATE EXTENSION postgis;"
* postgis_topology:psql -d test -c "CREATE EXTENSION postgis_topology;"
