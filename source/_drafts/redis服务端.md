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
