---
title: linux命令-find
date: 2018-02-26 16:17:49
tags: ['find']
categories: ['command']
---
# 功能说明
在指定的目录下搜索特定的文件

# 语法格式
find  path  option

# 选项参数
* -maxdepth n 最大查找深度【0为目录本身；1为当前目录下，依次类推；此选项需紧跟查找路径之后】
* -name 按照文件名查找文件【iname忽略大小写】
    - 当name后使用通配符*时必须将匹配模式使用引号括起来或进行转义
    - find . -name '*.c' -print 或 $ find . -name \*.c -print
* -size [±]n按大小匹配（大于、小于或等于n）
    - find . -maxdepth 1 -size +5k
* -type c按照文件类型匹配（f、d、p、s、b、c、l）
* -mtime [±]n按文件最后修改时间【以天计数】
* -mmin [±]n按文件最后修改时间【以分钟计数】
* -o 用于多个条件表达式的"或者"
* -a 用于多个条件表达式的"同时" 
    - find / -type f -size +50k -a -size -55k 2> /dev/null | xargs ls -lh | head -3
    - 查找根目录下大于50k小于55k的文件
* ! 逻辑取反
    - find / -type f -size +50k -a ! -user root
    - 查找根目录下大于50k且不属于root的文件

# 应用
## name的使用
* 表达式：find . -name *.log -mtime +60   
* 错误：paths must precede expression
* 原因：*在shell下会直接进行路径匹配
* 正确做法：find . -name '*.c' -print 或 $ find . -name \*.c -print

## 与xargs的搭配
find . -maxdepth 0 -type d -name "*8056" -mtime +119|xargs rm -rf {}
