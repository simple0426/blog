---
title: redis数据操作
tags:
categories:
---
# 批量删除key
## 删除所有数据
* 删除当前库的所有key：flushdb
* 删除所有数据库的key：flushall

## 删除特定key
* 所有匹配：redis-cli keys '*'|xargs -I {} redis-cli del {}
* 特定匹配：redis-cli keys 'fxx*'|xargs -I {} redis-cli del {}

# 数据类型
* 列表
    - 添加：rpush message wo
    - 查看：lrange message 0 2
* k/v
    - 设置：set he 2
    - 获取：set he 2
* HyperLogLog
    - 应用参考：https://www.cnblogs.com/ysuzhaixuefei/p/4052110.html
