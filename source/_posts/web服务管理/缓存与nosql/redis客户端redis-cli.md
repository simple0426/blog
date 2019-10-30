---
title: redis客户端redis-cli
tags:
  - redis-cli
categories:
  - redis
date: 2019-10-22 21:27:30
---

# 非交互模式
* 命令行使用：redis-cli incr mycounter
    - 重定向输出：redis-cli incr mycounter > /tmp/output.txt
    - 额外信息控制：
        + --raw：只显示命令结果
        + --no-raw：显示操作的数据类型及操作结果【这是默认模式，但是客户端也会判断如果有tty终端则显示数据类型，没有tty（重定向）则不显示数据类型】
* 连接
    - -h：连接主机
    - -p：连接端口，默认6379
    - -a：连接密码
    - -n：指定指定连接的库【默认连接数据库0】
    - -u：使用redis协议通过URI连接，如：redis-cli -u redis://password@redis-16379.hosted.com:16379/0 ping
* 重定向输入：
    - -x：读取最后一个参数：redis-cli -x set foo < /etc/services
    - 执行文件内的命令：cat /tmp/commands.txt | redis-cli
    ```
    set foo 100
    incr foo
    append foo xxx
    get foo
    ```
* 重复执行命令
    - -r：执行次数
    - -i：时间间隔，单位是秒，最小为0.1，默认为0
* 批量插入：--pipe【cat data.txt | redis-cli --pipe】
* 导出csv格式：--csv【redis-cli --csv lrange mylist 0 -1】

# 交互模式
>默认可以浏览历史命令、命令补全功能

- 选择数据库：select 2
- 数据库大小：dbsize
- 认证：auth
- 连接其他实例：connect
- 执行命令n次：n command
- 帮助信息：
    + 分类帮助：`help @<category>`
        * @generic
        * @list
        * @set
        * @sorted_set
        * @hash
        * @pubsub
        * @transactions
        * @connection
        * @server
        * @scripting
        * @hyperloglog
    + 特定命令帮助：help command
- 清屏：clear

# 特殊模式
* 状态监控
    - 参数：--stat （-i interval）
    - 范例：redis-cli --stat
* 数据库中扫描出最大key
    - 参数：--bigkeys
    - 范例：redis-cli --bigkeys
* 获取key列表
    - 参数：--scan【对比交互模式下`keys *`，scan参数以非阻塞方式获取key列表】
    - 范例：redis-cli --scan | head -10
    - 模式匹配参数：--pattern
    - 模式匹配范例：`redis-cli --scan --pattern '*-11*'`
* 发布订阅模式
    - 订阅：redis-cli psubscribe '*'
    - 发布：redis-cli PUBLISH mychannel mymessage
* 监控实例执行的命令：redis-cli monitor
* 用不同方式检测服务端实例延迟
    - --latency：每秒100次向服务端发送ping信号
    - --latency-history：每15秒开启一个新的会话连接用于压测【-i指定时间间隔】
    - --latency-dist：以带颜色的光谱图形式显示延迟信息
* 检测客户端延迟： --intrinsic-latency 【test-time】：redis-cli客户端所在主机的固有延迟【如内核调度，虚拟化开销等】
* 远程备份rdb
    - 参数：--rdb
    - 范例：redis-cli -h 172.15.22.9 -a 123456 --rdb /tmp/dump.rdb
* 测试复制模式下从机接收的内容
    - 参数：--slave
    - 范例：redis-cli -h 172.15.22.9 -a 123456 --slave
* 测试不同LRU策略下的key命中率
    - 影响因素：测试样本数量、最大可用内存(maxmemory )
    - 参数：--lru-test sample-number
    - 范例：./redis-cli --lru-test 10000000

[rediscli]: https://redis.io/topics/rediscli
