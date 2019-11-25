---
title: mongodb部署建议
tags:
categories:
---
# 存储引擎
* 3.2之后增加存储引擎WiredTiger
* 4.2之后移除对MMAPv1引擎的支持
* 若要将数据从MMAPv1迁移到WiredTiger，参考官方文档

# 版本支持
参考官方文档，确认不同操作系统不同版本支持的软件版本

# 数据目录dbPath
* 数据目录下包含的文件必须和存储引擎匹配，否则无法启动
* mongod程序必须 有dbPath目录的读写权限

# 并发
WiredTiger存储引擎默认支持document级别锁（类似关系型数据库的行级锁）

# 数据一致性
## Journal
* 使用Journal，即预写式日志保证数据可靠性
* 在4.0之后的版本，使用WiredTiger引擎、开启主从复制时不能关闭日志功能（storage.journal.enabled: false）

# NUMA使用
## 介绍
现在的机器上都是有多个CPU和多个内存块的。以前我们都是将内存块看成是一大块内存，所有CPU到这个共享内存的访问消息是一样的。这就是之前普 遍使用的SMP模型。  
但是随着处理器的增加，共享内存可能会导致内存访问冲突越来越厉害，且如果内存访问达到瓶颈的时候，性能就不能随之增加。NUMA（Non-Uniform Memory Access）就是这样的环境下引入的一个模型。  
 比如一台机器是有2个处理器，有4个内存块。我们将1个处理器和两个内存块合起来，称为一个NUMA node，这样这个机器就会有两个NUMA node。在物理分布上，NUMA node的处理器和内存块的物理距离更小，因此访问也更快。  
 比如这台机器会分左右两个处理器（cpu1, cpu2），在每个处理器两边放两个内存块(memory1.1, memory1.2, memory2.1,memory2.2)，这样NUMA node1的cpu1访问memory1.1和memory1.2就比访问memory2.1和memory2.2更快。所以使用NUMA的模式如果能尽量保证本node内的CPU只访问本node内的内存块，那这样的效率就是最高的。
## 缺点
当你的服务器还有内存的时候，发现它已经在开始使用swap了，甚至已经导致机器出现停滞的现象。这个就有可 能是由于numa的限制，如果一个进程限制它只能使用自己的numa节点的内存，那么当自身numa node内存使用光之后，就不会去使用其他numa node的内存了，会开始使用swap，甚至更糟的情况，机器没有设置swap的时候，可能会直接死机！所以你可以使用numactl --interleave=all来取消numa node的限制。
## 使用建议
* 如果你的程序是会占用大规模内存的，你大多应该选择关闭numa node的限制。因为这个时候你的程序很有几率会碰到numa陷阱。  
* 如果你的程序并不占用大内存，而是要求更快的程序运行时间。你大多应该选择限制只访问本numa node的方法来进行处理  

## mongodb使用
mongod="numactl --interleave=all /usr/local/mongod/bin/mongod"

# 其他设置
* 连接池【客户端设置】
* 内存：默认使用50% of (RAM - 1 GB)或256 MB中的大的数值
* cpu：默认的线城池数量和cpu核心数相关

# 磁盘和存储
* 系统设置swap分区，并设置swap【低度使用swap】
    - cat /proc/sys/vm/swappiness
    - sysctl vm.swappiness=1
* 文件系统
    - 尽量使用xfs文件系统
    - 不建议使用NFS等远程文件系统
    - 使用的文件系统支持目录级别fsync()【HGFS、virtualbox的共享目录不支持此操作】
* 建议使用RAID-10. RAID-5 
* 将数据、预写日志、日志、索引存放在不同磁盘以改善性能，mongodb支持目录的软链接

# 架构
* 副本集
* 分片集群

# 数据压缩存储
* snappy：默认压缩方式，压缩率较低，cpu使用也低
* zlib：压缩率较高，cpu使用也高
* zstd：4.2版本出现的压缩选项，在压缩率和cpu中折中选择

# 时钟同步
* 目的：为了保证mongodb组件或集群成员之间保证正常工作，应该开启ntp服务以同步主机时间
* 错误示例：【waited 189s for distributed lock configUpgrade for upgrading config database to new format v5】

# 系统参数
* 设置文件描述符：ulimit
* 禁用Transparent Huge Pages
* 禁用NUMA选项
* 关闭selinux
