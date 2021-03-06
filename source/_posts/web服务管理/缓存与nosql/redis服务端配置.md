---
title: redis服务端配置
tags:
  - redis.conf
  - 慢日志
categories:
  - redis
date: 2019-10-22 21:26:56
---

# 服务端配置方法
* 文件指令格式：keyword argument1 argument2 ... argumentN
    - 使用引号处理包含的空格：requirepass "hello world"
* 命令行传参：./redis-server --port 6380 --slaveof 127.0.0.1 6379
* 运行状态下变更配置：
    - 获取配置：config get keyword
    - 设置参数：config set keyword value
    - 将运行配置写入文件：config rewrite

# 专项配置
## 慢日志
### 格式
* 日志条目id
* 日志记录时间戳
* 执行时间（微秒）
* 执行命令数组
* 客户端ip和端口（4.0版本）
* 通过client setname设置的客户端名称（4.0版本）

### 配置参数
* slowlog-log-slower-than：执行时间阈值
* slowlog-max-len：记录日志条数

### 控制命令
* 查看（指定数目）日志：slowlog get （number）
* 查看日志记录总数：slowlog len
* 清空日志：slowlog reset

# 配置文件redis.conf
```
# redis.conf必须作为第一个参数，例如：./redis-server /path/to/redis.conf

# 涉及内存时的计量单位
# 1k => 1000 bytes
# 1kb => 1024 bytes
# 1m => 1000000 bytes
# 1mb => 1024*1024 bytes
# 1g => 1000000000 bytes
# 1gb => 1024*1024*1024 bytes

################################## INCLUDES ###################################

# include包含的文件不能被【config rewrite】命令改写
# 针对相同的配置项，后面的配置会覆盖前面的配置，所以需要注意include的位置【首行？尾行】
# include /path/to/local.conf

################################## MODULES #####################################

# 加载自定义模块
# loadmodule /path/to/my_module.so

################################## NETWORK #####################################

# 如果没有指定bind参数，redis运行在所有网络接口上
# redis可以指定一个或多个网络接口
# bind 192.168.1.100 10.0.0.1
# bind 127.0.0.1 ::1
bind 127.0.0.1

# 公网保护模式（默认开启）
# 如果没有配置bind指令或密码认证信息，则只能从127.0.0.1等回环接口访问
protected-mode yes

# 监听端口（默认6379，设置为0，则不在tcp上监听）
port 6379

# 定义tcp队列长度
# 在linux环境下，linux内核会自动将其值变更为/proc/sys/net/core/somaxconn的值，
# 以确保可以通过提高somaxconn的值达到最终目的
tcp-backlog 511

# 设置unix socket文件（默认不产生socket文件）
# unixsocket /tmp/redis.sock
# unixsocketperm 700

# 客户端连接超时时间
timeout 0

# tcp连接心跳检测（3.2.1之后，默认值为300s）
tcp-keepalive 300

################################# GENERAL #####################################

# 是否开启后台运行模式（默认no）
# 如果设置为后台模式，则会产生一个pid文件
daemonize no

# 如果使用其他控制程序(如upstart、systemd)控制redis服务启停，需要如下相关设置
#   supervised no      - no supervision interaction
#   supervised upstart - signal upstart by putting Redis into SIGSTOP mode
#   supervised systemd - signal systemd by writing READY=1 to $NOTIFY_SOCKET
#   supervised auto    - detect upstart or systemd method based on
#                        UPSTART_JOB or NOTIFY_SOCKET environment variables
# Note: these supervision methods only signal "process is ready."
#       They do not enable continuous liveness pings back to your supervisor.
supervised no

# 后台运行时产生的pid文件位置（如果没有指定，默认写入/var/run/redis.pid）
pidfile /var/run/redis_6379.pid

# 日志级别
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably)
# warning (only very important / critical messages are logged)
loglevel notice

# 日志文件位置
logfile ""

# 是否使用系统的syslog方式写日志
# syslog-enabled no

# Specify the syslog identity.
# syslog-ident redis

# Specify the syslog facility. Must be USER or between LOCAL0-LOCAL7.
# syslog-facility local0

# 设置数据库数量，每个库都有独立的命名空间
databases 16

# 交互模式下，是否显示redis logo信息
always-show-logo yes

################################ SNAPSHOTTING  ################################

# 设置保存数据到rdb文件的频率：save <seconds> <changes>
# 取消所有保存频率的设置：save ""
# 
# 以下设置的含义：
# 900秒后至少有1个key发生变更
# 300秒后至少有10个key发生变更
# 60秒后至少有10000个key发生变更
save 900 1
save 300 10
save 60 10000

# 开启rdb写入时，如果后台持久化存储失败则停止客户端写操作
stop-writes-on-bgsave-error yes

# rdb文件压缩存储
rdbcompression yes

# rdb校验【增加可用性，但会增加存储或加载开销】
rdbchecksum yes

# rdb文件名
dbfilename dump.rdb

# 指定rdb和aof文件的存储目录
dir ./

################################# REPLICATION #################################

# 主从复制从机设置
# slaveof <masterip> <masterport>

# 主机要求的验证密码（requirepass）
# masterauth <master-password>

# 主从复制中断时，从机的处理（默认yes）：
# 设置为yes，则从机可以使用过期的数据为客户端提供服务
# 设置为no，则从机会对除了INFO和SLAVEOF的其他命令返回错误信息："SYNC with master in progress"
slave-serve-stale-data yes

# 从机只读设置
slave-read-only yes

# rdb文件传输方式（复制同步策略）：
# 有盘方式：主开启写进程在本地写rdb文件，在rdb文件阶段性写完后，
#   就可以和（多个）从机通信，然后由父进程传输文件到从机
# 无盘方式（网络）：主开启写进程直接将rdb文件写入从机网络socket，
#   当阶段性写操作完成后，主和另一个从机通信进行复制操作
#
# 无盘复制同步要点：
# 1.无盘模式当前是实验性功能
# 2.新建一个从机或从机重连时会执行一个全量同步
# 3.使用无盘方式，会配置一个等待时间，以便所有从机可达以并行方式传输
# 4.磁盘容量小或磁盘io小，同时网络带宽大的情况更适合使用无盘方式
repl-diskless-sync no

# 无盘方式工作时，复制开始前主机的等待时间（默认5s，0则不等待）
# 一旦同步开始，新的从机并不会被主接纳，所以需要配置一个等待时间，确保所有从机已连上。
repl-diskless-sync-delay 5

# 从机检测主机可达的时间间隔
# repl-ping-slave-period 10

# 复制超时时间
# 它包含：主机视角的ping响应超时、从机角度的ping和数据传输超时
# 它必须比repl-ping-slave-period时间长
# repl-timeout 60

# 是否关闭tcp nodelay功能（默认no，不关闭延迟）
# 设置为yes，会传输更小的数据包并使用更小的带宽，但延迟会增加（linux内核限制，最多40ms）
# 设置为no，延迟会更低，但是需要更大的带宽
repl-disable-tcp-nodelay no

# 允许从机断开并重连时可以继续同步的缓冲区大小【避免全量同步】
# repl-backlog-size 512mb

# 超时则主释放同步缓冲区，0则永不释放
# repl-backlog-ttl 3600

# 从机提升为主机时的优先级（数越小优先级越高）
# 主要是redis sentinel使用，当主出现异常时提升从机为主
slave-priority 100

# 当少于指定数量的从机或复制延迟大于某个阈值时，主停止写操作（设置为0则关闭此功能）
# 范例：至少3个从机的延迟小于10，否则master停止写入操作
# min-slaves-to-write 3
# min-slaves-max-lag 10

# 主从复制时，主查看从机ip的方式：INFO replication 和 ROLE
# 在端口转发和nat网络情况下，上述命令获取的信息会有误
# 通过在从机声明如下变量，可以告诉主机自己真实的ip和port【可选设置】
# slave-announce-ip 5.5.5.5
# slave-announce-port 1234

################################## SECURITY ###################################

# 客户端认证密码（也是主从复制的密码），由于redis没有防爆破的手段，所以需要设置一个强密码
# requirepass foobared

# 重命名命令：
# rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
# 也可通过空字符串“删除”命令
# rename-command CONFIG ""
# 【注意】重命名命令在写入aof或传输到从机时会出问题

################################### CLIENTS ####################################

# 客户端最大连接数
# 如果redis不能设置最大文件描述符数量，
#   则maxclients被设置为当前最大文件描述符数量减32（redis保留部分描述符供内部使用）
# maxclients 10000

############################## MEMORY MANAGEMENT ################################

# 最大可用内存（超过后根据驱逐策略删除key）
# 1. 超过此阈值，如果驱逐策略为noeviction，则对于写命令则返回错误，读命令则正常返回
# 2. 作为LRU或LFU缓存，或使用noeviction驱逐策略时，都应该设置硬限制
# 3. 不包含主从复制的输出缓冲区大小（repl-backlog-size）
# maxmemory <bytes>

# 内存使用达到限制时的key删除策略（默认noeviction）：
# volatile-lru -> 使用LRU策略驱逐具有过期设置的key
# allkeys-lru -> 使用LRU策略驱逐任意的key
# volatile-lfu -> 使用LFU策略驱逐具有过期设置的key
# allkeys-lfu -> 使用LFU策略驱逐任意的key
# volatile-random -> 从具有过期设置的key中随机删除一个
# allkeys-random -> 从所有key中随机删除一个
# volatile-ttl -> 删除最接近过期的key
# noeviction -> 不驱逐任何key，向写操作返回错误
#
# LRU：Least Recently Used【最近最少使用】
# LFU：Least Frequently Used【经常最少使用】
# LRU, LFU 和 volatile-ttl都使用了近似的随机算法
#
# 即使使用上述策略，如果没有合适的key被删除，依然会对写操作返回错误，包含的写操作如下：
# set setnx setex append
# incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
# sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
# zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
# getset mset msetnx exec sort
#
# maxmemory-policy noeviction

# 启动内存回收时，单次取样数量：
# 3个速度最快，但是策略执行不精确；5个足够；10个最精确，但是更消耗cpu
# maxmemory-samples 5

############################# LAZY FREEING ####################################

# 非阻塞式释放内存
#
# redis有两种删除key的模式：
# 1.阻塞式：del命令是一种阻塞式删除，即在进行删除操作的同时停止处理新命令，阻塞时间的长短根据删除对象的大小而定
# 2.非阻塞式：redis（4.0.0之上版本）也提供非阻塞式删除命令，比如unlink（非阻塞del）、flushdb|flushall的async选项
# 这些命令会在限定时间内执行完，但是会在后台开启一个新线程持续释放内存空间
#
# 情景设置：
# 一、在下列情景中，redis默认会以阻塞方式删除key或清空库：
# 1.根据最大内存和驱逐策略删除key
# 2.由于key的过期设置（EXPIRE）
# 3.对一个已存在的key进行操作，如RENAME
# 4.在复制操作时，从机执行全量复制时，会清空从机rdb
# 
# 二、可以单独设置以上情况以非阻塞方式释放内存，具体命令如下：
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
slave-lazy-flush no

############################## APPEND ONLY MODE ###############################

# 是否开启预写日志模式（aof和rdb可以同时开启）
# 默认情况下，redis异步导出数据到磁盘（基于rdb的不同刷新策略（save）可能会有部分数据丢失）
# 当aof开启时，redis启动时会将加载aof文件
appendonly no

# aof文件名
appendfilename "appendonly.aof"

# fsync为数据写入磁盘的策略
# no: 让操作系统处理什么时候刷新数据到磁盘，此时速度最快
# always: 每次都等待数据写入aof文件，速度最慢，安全性最好
# everysec: 每秒刷新数据到磁盘，折中选择【默认选项】
appendfsync everysec

# 在后台执行BGSAVE或BGREWRITEAOF命令时，是否阻止主进程进行磁盘刷新操作
# 这个选项是为了减轻磁盘写压力，但会造成数据丢失【默认不开启】
no-appendfsync-on-rewrite no

# 自动重写aof文件时，增长的大小所占的百分比
auto-aof-rewrite-percentage 100
# 自动重写aof文件时，最小增加的文件大小
auto-aof-rewrite-min-size 64mb

# 是否允许加载不完整（被截断）的aof文件（如果设置为no，必须使用redis-check-aof修复aof文件）
aof-load-truncated yes

# 使用aof和rdb混合方式持久化数据【可以更快的执行重写和恢复操作，默认关闭】
aof-use-rdb-preamble no

################################ LUA SCRIPTING  ###############################

# lua脚本执行超时时间（毫秒，0或负值则不限制时间）
# 超过时间，redis会记录脚本依然在执行，并对查询信息返回错误
# 【SCRIPT KILL】命令会停止一个不含写操作的脚本
# 【SHUTDOWN NOSAVE】可以停止包含写操作的脚本
lua-time-limit 5000

################################ REDIS CLUSTER  ###############################

# 是否开启redis集群功能
# redis单实例模式和集群模式不能相互转换，只能在一开始就确定使用的方式
# cluster-enabled yes

# 集群节点的配置文件（集群自动创建和更新，一般不需要手动编辑）每个节点都有一个配置文件
# cluster-config-file nodes-6379.conf

# 节点断开超时时间（毫秒，其他内部限制时间都是这个值的倍数）
# 超过这个时间，节点被认为是失败状态
# cluster-node-timeout 15000

# 定义从机选举为主的因子（避免数据太旧的从机执行故障切换）
# 
# 数据新旧情况判断：
# 1.当有多个从机时：会根据他们从主上接收数据的多少（offset）选择谁执行故障切换
# 2.单个从机角度计算：则根据距离上次接收ping响应、接收命令的时间（连接状态下）或与主断开的时间
# 
# 第2点可以由用户控制，超过【(node-timeout * slave-validity-factor) + repl-ping-slave-period】
# 的从机不能提升为主；其中slave-validity-factor是从机选举为主的因子：
# 1）定义为n，则超过（n*timeout+ping）后选举为主，此段时间内，集群不可用除非原来的主恢复
# 2）如果：非常大的值允许有非常旧数据的从机执行故障切换，值太小则无法在众多从机中选出提升为主的从机
# 3）默认为0，超时即选为主；此设置保证了最大可用性，允许从机立即提升为主，但是依然会根据从主机接收数据的多少选择提升为主的从机
#
# cluster-slave-validity-factor 10

# 主机在有如下数量的从机时，其他从机可以自动迁移到孤立的主机（没有从机的主机）
# cluster-migration-barrier 1

# 是否要求所有数据槽都可用时集群才可用
# 设置为no时，允许集群数据不完整时依然可以提供查询服务
# cluster-require-full-coverage yes

# 主故障时，禁止从机执行故障切换（此时会手动处理主机故障切换操作）
# 在有多个数据中心时非常有用，此时会期望只有这个数据中心的节点全部出现故障时才执行切换
# cluster-slave-no-failover no

########################## CLUSTER DOCKER/NAT support  ########################

# 当遇到nat网络或端口转发时【如docker等容器环境】集群节点地址发现会失败，所以
# 每个集群节点都需要知道一个公共地址，即如下配置：
# * cluster-announce-ip
# * cluster-announce-port
# * cluster-announce-bus-port
#
# Example:
#
# cluster-announce-ip 10.1.1.5
# cluster-announce-port 6379
# cluster-announce-bus-port 6380

################################## SLOW LOG ###################################

# 慢日志在内存中记录超过一定执行时间的命令：
# 不包含io操作，比如：和客户端通信、发送响应
# 仅包含：命令的实际执行时间，此时线程被阻塞不能服务于其他请求

# 执行时间阈值（微秒，负值则关闭slowlog功能，0则记录每一个命令）
slowlog-log-slower-than 10000

# 保存多少条慢日志，超过则删除最旧的记录（【SLOWLOG RESET】强制回收内存）
slowlog-max-len 128

################################ LATENCY MONITOR ##############################

# 超过阈值的延迟因素会被记录（毫秒；默认为0，关闭功能 ）
latency-monitor-threshold 0

############################# EVENT NOTIFICATION ##############################

# redis可以通知发布/订阅的客户端发生在键空间的事件，
# 相关文档：http://redis.io/topics/notifications
#
# redis通过单字符定义要通知的事件：
#  K     Keyspace events, published with __keyspace@<db>__ prefix.
#  E     Keyevent events, published with __keyevent@<db>__ prefix.
#  g     Generic commands (non-type specific) like DEL, EXPIRE, RENAME, ...
#  $     String commands
#  l     List commands
#  s     Set commands
#  h     Hash commands
#  z     Sorted set commands
#  x     Expired events (events generated every time a key expires)
#  e     Evicted events (events generated when a key is evicted for maxmemory)
#  A     Alias for g$lshzxe, so that the "AKE" string means all the events.
# 
# notify-keyspace-events使用0或多个字符构成的字符串串作为参数：
# 1. 0个字符串表示关闭事件通知功能（默认）
# 2. 至少需要指定K或E，否则会没有事件通知
#
# 范例1：开启列表和通用事件，从事件名称的角度，使用如下
#  notify-keyspace-events Elg
#
# 范例2：获取提交给频道<__keyevent@0__:expired>的过期key流
#  notify-keyspace-events Ex
notify-keyspace-events ""

############################### ADVANCED CONFIG ###############################

# ziplist是一个解决空间紧凑的数据存储结构，但是当数据超过此阈值时，将采用原生的数据存储结构
# 
# 数据类型为hash，设置使用ziplist的条目数量、最大条目值的阈值
hash-max-ziplist-entries 512
hash-max-ziplist-value 64

# 数据类型是list，设定使用ziplist的阈值
# 定义为正值时，表示单个列表最大元素数量
# 定义为负值时（-5到-1）表示单个列表的最大存储空间
# -5: max size: 64 Kb  <-- not recommended for normal workloads
# -4: max size: 32 Kb  <-- not recommended
# -3: max size: 16 Kb  <-- probably not recommended
# -2: max size: 8 Kb   <-- good
# -1: max size: 4 Kb   <-- good
list-max-ziplist-size -2

# 数据类型是list，也可以被压缩存储
# 压缩深度定义如下
# 0：关闭列表所有节点的压缩
# 1：列表的首尾节点不压缩，其他节点都压缩
# 2：列表的首及相邻节点、尾及相邻节点不压缩，其他节点都压缩
# n：以此类推，首尾的n个节点不压缩，中间节点压缩
list-compress-depth 0

# 数据类型为set，且仅由基数为10的整数（64为有符号整数）组成的字符串
# 当set类型的大小小于阈值时，可以使用这种节省内存的编码
set-max-intset-entries 512

# 数据类型sorted sets（zset），设定使用ziplist的阈值
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

# HyperLogLog的稀疏表示限制（建议值是3000，取值范围是0到15000）
# 1. 大于这个阈值，数据使用稠密数据结构；
# 2. 小于此值，数据使用稀疏数据结构。
# 3. 大于16000的值是无意义的，因为此时稠密表示法更能有效使用内存
hll-sparse-max-bytes 3000

# 是否开启对主字典的二次哈希（hash会释放内存，但因为使用cpu会造成微量延迟）
activerehashing yes

# 客户端输出缓冲区限制，如果客户端读取数据的速度不够快，则强制断开
# 参数0不限制；一般的客户端（normal）不限制，因为他们使用应答模式返回数据
# 
# 语法：
# client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
#
# redis：定义了3种不同类型的客户端
# normal -> normal clients including MONITOR clients
# slave  -> slave clients
# pubsub -> clients subscribed to at least one pubsub channel or pattern
# 
# 如果客户端使用达到硬限制或软限制（同时达到字节大小和持续时间）则断开客户端
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# 客户端查询缓冲区限制（如果有特殊需求可以设置）
# client-query-buffer-limit 1gb

# 使用redis协议时，单个大数据量请求的字符串长度限制（默认512mb）
# proto-max-bulk-len 512mb

# redis调用内部函数执行后台任务（如：超时关闭客户端，删除过期的key）的频率：
# 1. 增大数值可以更精确的处理各类超时、删除过期key等问题，但是会消耗更多的cpu、也会增加redis的延迟
# 2. 取值范围1到500，但是不建议使用超过100的值
hz 10

# 当aof文件重写时，每产生32MB数据就会执行一次刷新磁盘操作
# 以这种渐进的方式进行重写操作可以降低磁盘写压力（从而降低常规写aof的延迟）
aof-rewrite-incremental-fsync yes

# 【LFU算法调节】
# 可调节参数：计数器对数因子（lfu-log-factor）和计数器衰减时间（lfu-decay-time）
#
# 【计数器对数因子-原理】
# 针对每个key，LFU计数器只有8位，最大值是255；
# 给旧的计数器一个值，当key被访问时，计数器以下面方式增长
# 1.产生一个0~1之间的随机数R
# 2.计算一个概率值P：1/(old_value*lfu_log_factor+1)
# 3.只有当概率值P大于随机数R时，计数器才递增
#
# 下边是不同对数因子在不同访问次数下的频率计数器变化
# （计数器的初试值为5，以便计算对象的累积命中率；测试命令如下）：
# 1. redis-benchmark -n 1000000 incr foo
# 2. redis-cli object freq foo
# +--------+------------+------------+------------+------------+------------+
# | factor | 100 hits   | 1000 hits  | 100K hits  | 1M hits    | 10M hits   |
# +--------+------------+------------+------------+------------+------------+
# | 0      | 104        | 255        | 255        | 255        | 255        |
# +--------+------------+------------+------------+------------+------------+
# | 1      | 18         | 49         | 255        | 255        | 255        |
# +--------+------------+------------+------------+------------+------------+
# | 10     | 10         | 18         | 142        | 255        | 255        |
# +--------+------------+------------+------------+------------+------------+
# | 100    | 8          | 11         | 49         | 143        | 255        |
# +--------+------------+------------+------------+------------+------------+
#
# 【计数器衰减时间-原理】
# 必须是经过的时间，以便将key计数器除以2或直接递减（小于等于10时） 
#
# 【计数器对数因子-设置】（0-255）（因子越高，为了达到最大值，需要更多的访问）
# 如下表示：每100万个请求使计数器饱和
# lfu-log-factor 10
#
# 【LFU衰减时间周期-设置】（分钟），默认1分钟；
# 如下表示：每分钟使计数器衰减一次
# lfu-decay-time 1

########################### ACTIVE DEFRAGMENTATION #######################

# 开启实时内存碎片整理功能
# activedefrag yes

# 内存碎片化达到多少字节后开启整理
# active-defrag-ignore-bytes 100mb

# 内存碎片化达到多大比率时开启整理
# active-defrag-threshold-lower 10

# 碎片整理的最大百分比
# active-defrag-threshold-upper 100

# 碎片整理占用的最小cpu使用率
# active-defrag-cycle-min 25

# 碎片整理占用的最大cpu使用率
# active-defrag-cycle-max 75
```
