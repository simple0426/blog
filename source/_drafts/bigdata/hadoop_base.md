# 大数据  
__与数据的规模无关__  

* 对数据的处理程度  
* 对数据的处理速度  

# 大数据产品  
* hadoop -->第一代  
* spark  -->第二代  

# hadoop的核心  
## 存储[hdfs]
* namenode[主节点，只有一个]
    * 接收用户操作请求
    * 维护文件系统的目录结构
    * 管理文件和block以及block和datanode之间的关系
* datanode[从系欸但那，有很多个]
    * 存储文件  
[文件被分成block存储在磁盘上，为保证数据安全，文件会有多个副本]  

## 计算框架
* map/reduce
    * 并行计算
    * 基于磁盘读写
* storm
    * 流处理
* hbase
    * nosql处理
* hive
    * sql处理  

## 集群资源管理和调度[yarn]
* master
    * resource manager --》总的资源管理
* node
    * mrapp master --》负责某个计算框架job的任务调度
    * yarnchild --》执行mrapp master分配的计算任务并汇报结果

## 处理速度[cpu]
