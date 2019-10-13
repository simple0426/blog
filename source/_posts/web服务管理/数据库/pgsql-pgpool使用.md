---
title: pgsql-pgpool使用
tags:
  - pgpool
categories:
  - pgsql
date: 2019-10-13 17:58:10
---

# pgpool简介
pgpool是一个位于数据库服务端和客户端之间的中间件，可以实现如下功能：

* 连接池
* 复制功能：可以实时的把数据备份到单独的磁盘
* 负载均衡：将查询语句拆分到多台后端数据库
* 限制过量的连接：将超过限制的连接排队，但不直接返回错误
* 看门狗功能：同时在两个节点部署pgpool，通过vip偏移执行高可用
* 基于内存的查询缓存

# 软件安装
./configure --prefix=/usr/local/pgpool --with-pgsql=/usr/local/pgsql/
make
make install

# 安装插件
## pgpool_recovery
$ cd pgpool-II-x.x.x/src/sql/pgpool-recovery
$ make
$ make install
$ psql -f pgpool-recovery.sql template1
$ 【vim postgresql.conf】：pgpool.pg_ctl = '/usr/local/pgsql/bin/pg_ctl'
$ pg_ctl reload -D /usr/local/pgsql/data

## pgpool-regclass
 PostgreSQL 9.4 or later可以省略

## insert_lock表
如果打算使用原生的复制模式和insert_lock，需要创建：
$ cd pgpool-II-x.x.x/src/sql
$ psql -f insert_lock.sql template1

# 配置[pgpool.conf][pgpool-conf]
```
#连接
listen_addresses = '*'
port = 9833
pcp_listen_addresses = '*'
pcp_port = 9898

#后端
backend_hostname0 = '10.86.0.10'
backend_port0 = 5432
backend_weight0 = 16
backend_data_directory0 = '/home/postgres/db0/db/9.3/data/'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_hostname1 = '10.86.0.12'
backend_port1 = 5432
backend_weight1 = 128
backend_data_directory1 = '/home/postgres/db0/db/9.3/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'

#安全
enable_pool_hba = on
pool_passwd = ''
authentication_timeout = 60
ssl = off

#连接池
num_init_children = 64
max_pool = 16
child_life_time = 300
child_max_connections = 200
connection_life_time = 120
client_idle_limit = 60
connection_cache = on
reset_query_list = 'ABORT; DISCARD ALL'
relcache_expire = 0
relcache_size = 256
check_temp_table = on

#日志
log_destination = 'syslog'
print_timestamp = on
log_connections = off
log_hostname = off
log_statement = off
log_per_node_statement = off
log_standby_delay = 'none'
syslog_facility = 'LOCAL0'
syslog_ident = 'pgpool'
debug_level = 0

#文件位置
pid_file_name = '/home/postgres/pgpool2/run/pgpool.pid'
logdir = '/home/postgres/pgpool2/log/'

#基于流复制的负载均衡模式（主备模式）
replication_mode = off
insert_lock = on
load_balance_mode = on
master_slave_mode = on
master_slave_sub_mode = 'stream'
ignore_leading_white_space = on
white_function_list = ''
black_function_list = 'nextval,setval'
sr_check_period = 3
sr_check_user = 'replication'
sr_check_password = 'repl-pg-0232'
delay_threshold = 0

#健康检查
health_check_period = 10
health_check_timeout = 20
health_check_user = 'postgres'
health_check_password = 'YUTR0-Lew&4-DE74GF-qePi8'
health_check_max_retries = 0
health_check_retry_delay = 3

#故障切换
failover_command = '/home/postgres/bin/failover_stream.sh %d %H /home/postgres/db0/tmp/trigger_file'
failback_command = '/bin/rm -f /home/postgres/db0/tmp/trigger_file'
fail_over_on_backend_error = on #设置health_check_max_retries则关闭此选项
search_primary_node_timeout = 10

#在线恢复
recovery_user = 'postgres'
recovery_password = 'YUTR0-Lew&4-DE74GF-qePi8'
recovery_1st_stage_command = '/home/postgres/bin/basebackup.sh'
recovery_2nd_stage_command = ''
recovery_timeout = 90
client_idle_limit_in_recovery = 0

#双击热备配置
use_watchdog = on                   #   开启看门狗，用于监控pgpool 集群健康状态
wd_hostname = '10.86.0.10'             #   本地看门狗地址
wd_port = 9000                          #
wd_priority = 1                         #   看门狗优先级，用于pgpool 集群中master选举
delegate_IP = '10.86.0.100'             #    VIP 地址
if_up_cmd = 'ip addr add $_IP_$/24 dev eth0 label eth0:0'  # 配置虚拟IP到本地网卡
if_down_cmd = 'ip addr del $_IP_$/24 dev eth0'          #   
wd_lifecheck_method = 'heartbeat         # '  看门狗健康检测方法
wd_heartbeat_port = 9694                #     看门狗心跳端口，用于pgpool 集群健康状态通信
wd_heartbeat_keepalive = 2              #     看门狗心跳检测间隔
wd_heartbeat_deadtime = 30              #
heartbeat_destination0 = '10.86.0.12'  #     配置需要监测健康心跳的IP地址，非本地地址，即互相监控，配置对端的IP地址
heartbeat_destination_port0 = 9694      # 监听的端口
heartbeat_device0 = 'eth0'              # 监听的网卡名称
wd_life_point = 3               #    生命检测失败后重试次数
wd_lifecheck_query = 'SELECT 1' #   用于检查 pgpool-II 的查询语句。默认为“SELECT 1”。
wd_lifecheck_dbname = 'postgres'        # 检查健康状态的数据库名称
wd_lifecheck_user = 'pgcheck'           # 检查数据库的用户，该用户需要在Postgres数据库存在，且有查询权限
wd_lifecheck_password = '123456'        #   看门狗健康检查用户密码
other_pgpool_hostname0 = '10.86.0.12'  # 指定被监控的 pgpool-II 服务器的主机名
other_pgpool_port0 = 9999       # 指定被监控的 pgpool-II 服务器的端口号
other_wd_port0 = 9000           # 指定 pgpool-II 服务器上的需要被监控的看门狗的端口号
```

# 配置pcp.conf
pcp为pgpool管理接口，pcp.conf里包含有管理接口的认证信息，内容如下：
```
# USERID:MD5PASSWD
postgres:e10adc3949ba59abbe56e057f20f883e
```
md5密码产生：pg_md5 123456

# 配置pool_hba.conf
* pool_hba和pg_hba类似，是客户端连接pgpool的认证
* 当pgpool使用pg原生模式或只有一个后端服务器时，可以不使用pool_hba
* 配置：
    - 从数据库中查询用户及密码：select rolpassword from pg_authid where rolname='pgtest'; 
    - 建立文件pool_passwd，并写入查询到的账号信息：【pgtest:md5adbe22045e9d4ebb5254ed06a0987528】

# 服务启停控制
* 启动服务：pgpool
    - 前台debug模式：pgpool -n -d
* 停止服务：pgpool 【-m smart|fast|immediate】stop
* 重载配置：pgpool reload

# 服务状态查看
sql命令行下执行如下命令：

* 节点状态：show pool_nodes;
* 进程池：show pool_processes;
* 配置信息：show pool_status;
* 连接池：show pool_status;

# 节点管理
* 卸载节点：pcp_detach_node -h 127.0.0.1 -p 9898 -n 1
* 添加节点：pcp_attach_node -h 127.0.0.1 -p 9898 -n 1
* 恢复节点为可用状态：pcp_recovery_node【配合pgpool.conf中的ONLINE RECOVERY配置】

# [故障自动切换配置][pgpool-failover]
* failover_command = '/usr/local/pgpool/failover.sh'
* failover.sh

```
ssh -T 172.16.0.215 "/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data/ -l /home/postgres/logfile promote"
```

[pgpool-conf]:https://www.pgpool.net/docs/latest/en/html/runtime-config.html
[pgpool-failover]: https://www.pgpool.net/docs/latest/en/html/runtime-config-failover.html
