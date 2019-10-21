---
title: redis服务端
tags:
categories:
---
# redis.conf
* 命令格式：keyword argument1 argument2 ... argumentN
* 使用引号处理包含的空格：requirepass "hello world"
* 命令行传参：./redis-server --port 6380 --slaveof 127.0.0.1 6379
* 运行状态下的配置变更：
    - 获取配置：config get keyword
    - 设置参数：config set keyword value
    - 将运行配置写入文件：config rewrite
* [将redis作为缓存使用](https://redis.io/topics/lru-cache)
    - maxmemory 2mb
    - maxmemory-policy allkeys-lru

# 慢日志
## 格式
* 日志条目id
* 日志记录时间戳
* 执行时间（微秒）
* 执行命令数组
* 客户端ip和端口（4.0版本）
* 通过client setname设置的客户端名称（4.0版本）

## 配置参数
* slowlog-log-slower-than：执行时间阈值
* slowlog-max-len：记录日志条数

## 控制命令
* 查看（指定数目）日志：slowlog get （number）
* 查看日志记录总数：slowlog len
* 清空日志：slowlog reset
