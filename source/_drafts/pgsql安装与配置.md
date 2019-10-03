---
title: pgsql安装与配置
tags:
categories:
---
# 依赖安装
* make
* gcc
* libreadline6 libreadline6-dev【用于sql命令行提示】
* zlib1g zlib1g-dev【备份和恢复时压缩】
* libpython2.7 libpython2.7-dev libpython3.5 libpython3.5-dev【python语言支持】
* OpenSSL【连接加密】
* libxslt1.1 libxslt1-dev【xlst与xml2支持】
* libxml2 libxml2-dev【添加xml支持】

# 安装
* 编译配置：./configure --with-python --with-openssl --with-libxml --with-libxslt
* 编译与安装：make && sudo make install

# 链接库设置
* 实时设置链接库【官方】
    - 【普通用户】LD_LIBRARY_PATH=/usr/local/pgsql/lib;export LD_LIBRARY_PATH
    - 【root】/sbin/ldconfig /usr/local/pgsql/lib
* 静态化设置：echo usr/local/pgsql/lib >> /etc/ld.so.conf;ldconfig

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
## 依赖安装
http://www.postgis.net/docs/postgis_installation.html#install_requirements
# 安装
./configure --with-pgconfig=/usr/lib/postgresql/9.3/bin/pg_config  --without-topology
sudo make
sudo make install