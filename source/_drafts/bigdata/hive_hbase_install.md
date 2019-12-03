# 软件下载
* [hive][1]
* [hbase][2]
* [mysql-connector][3]
* [zookeeper][4]

# zookeeper
>由于hbase依赖于Zookeeper，所以需要先安装Zookeeper  

* 解压Zookeeper
* 配置zoo.cfg

```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/home/test/app/zookeeper-3.4.6/
clientPort=2181
server.1=Bgdata03:2888:3888   
server.2=Bgdata04:2888:3888  
server.3=Bgdata05:2888:3888
#server.1,server.2,server.3和myid相匹配
```

* 复制

```
rsync -azvP -e ssh zookeeper-3.4.6 Bgdata05:~/app/
rsync -azvP -e ssh zookeeper-3.4.6 Bgdata04:~/app/
```

* 分主机创建文件->myid

```
cd /home/test/app/zookeeper-3.4.6
echo 1 > myid
# 其他主机分别为echo 2 > myid;echo 3 > myid
```

* 启动
  * 启动：`/zookeeper-3.4.6/bin/zkServer.sh start`
  * 状态查看：`./zookeeper-3.4.6/bin/zkServer.sh status`

# hbase
## 解压bin包
## 配置环境变量
```
export HBASE_HOME=/home/test/app/hbase-1.3.1
export PATH=$PATH:$HBASE_HOME/bin
```
## 配置文件修改
>cd hbase-1.3.1/;mkdir logs

* hbase-env.sh

```
export JAVA_HOME=/home/test/app/jdk1.8.0_121
export HADOOP_HOME=/home/test/app/hadoop-2.7.4
export HBASE_HOME=/home/test/app/hbase-1.3.1
export HBASE_MANAGES_ZK=false
export HBASE_LOG_DIR=/home/test/app/hbase-1.3.1/logs
```

* hbase-site.xml

```
<configuration>    
  <property>    
    <name>hbase.rootdir</name>    
    <value>hdfs://Bgdata03:9000/hbase</value>    
  </property>    
  <property>    
     <name>hbase.cluster.distributed</name>    
     <value>true</value>    
  </property>    
  <property>    
      <name>hbase.master</name>    
      <value>Bgdata03:60000</value>    
  </property>    
   <property>    
    <name>hbase.zookeeper.property.dataDir</name>    
    <value>/home/test/app/zookeeper-3.4.6</value>    
  </property>    
  <property>    
    <name>hbase.zookeeper.quorum</name>    
    <value>Bgdata03,Bgdata04,Bgdata05</value>    
  </property>    
  <property>    
    <name>hbase.zookeeper.property.clientPort</name>    
    <value>2181</value>    
  </property>   
  <property>  
  <name>hbase.master.info.port</name>  
  <value>60010</value>  
</property>   
</configuration>   
```

* regionservers

```
Bgdata03
Bgdata04
Bgdata05
```

## 复制
```
rsync -azvP -e ssh hbase-1.3.1 Bgdata04:~/app/
rsync -azvP -e ssh hbase-1.3.1 Bgdata05:~/app/
```
## 启动
`./bin/start-hbase.sh`

# hive
>由于hive操作的是mysql，所以需事前安装和配置mysql  

## 解压hive包并配置环境变量
```
export HIVE_HOME=/home/test/app/apache-hive-2.1.1-bin
export PATH=$PATH:$HIVE_HOME/bin
```

## 安装mysql驱动
将jar包防止在apache-hive-2.1.1-bin/lib/目录下
## 配置文件修改
* hive-env.sh

`export JAVA_HOME=/home/test/app/jdk1.8.0_121`

* hive-site.xml

```
<configuration>
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://172.17.134.67:3306/hive?characterEncoding=UTF-8</value>
  <description>JDBC connect string for a JDBC metastore</description>
</property>
<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>
  <description>Driver class name for a JDBC metastore</description>
</property>
<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>hive</value>
  <description>username to use against metastore database</description>
</property>
<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>hive_passwd</value>
  <description>password to use against metastore database</description>
</property>
</configuration>
```
## 复制
```
rsync -azvP -e ssh apache-hive-2.1.1-bin Bgdata05:~/app/
rsync -azvP -e ssh apache-hive-2.1.1-bin Bgdata04:~/app/
```
## 初始化
`schematool -initSchema -dbType mysql`
此时可以在数据库hive看到创建的表

## 启动
[非守护进程服务，类似语言环境]：`hive`

## 测试
1. hive终端创建表`CREATE TABLE xp(id INT,name string) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'`
2. linux终端创建文件/home/test/test.txt
    * 根据建表语句，id和name之间必须使用tab分隔
3. hive终端导入数据`load data local inpath '/home/test/test.txt' into table xp;`
4. 结果查询
    * hive终端查询导入结果`select * from xp;`
    * hadoop的web界面查询导入结果<http://127.0.0.1:50070/explorer.html#/user/hive/warehouse/xp>
    * mysql查询建表结果`select * from hive.TBLS;`

# 安装参考
* [Linux中基于hadoop安装hive][5]
* [分布式安装Hadoop,Hive,Hbase][6]


[1]:http://mirrors.aliyun.com/apache/hive/hive-2.1.1/apache-hive-2.1.1-bin.tar.gz
[2]:http://mirrors.aliyun.com/apache/hbase/1.3.1/hbase-1.3.1-bin.tar.gz
[3]:https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.44.tar.gz
[4]:http://mirrors.aliyun.com/apache/zookeeper/zookeeper-3.4.6/zookeeper-3.4.6.tar.gz
[5]: http://blog.csdn.net/pucao_cug/article/details/71773665
[6]: http://blog.csdn.net/clark_xu/article/details/69668618

