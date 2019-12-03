# 软件下载
* [hadoop][4]
* [java8][5]

# 主机初始化配置:[主从节点]  
1. 修改主机名，并使hosts一致  
2. 关闭防火墙/selinux/iptables  
3. 保持时间一致  
4. 每台主机都新建普通用户,并使master和node全互连免密码ssh登陆。[实现参考][3]

# java与hadoop安装[主节点]
* 解压[sun jdk](https://www.java.com/zh_CN/download/manual.jsp)和hadoop
* 设置环境变量[/etc/profile]  

```
export JAVA_HOME=/home/test/app/jdk1.8.0_121
export HADOOP_HOME=/home/test/app/hadoop-2.7.4
export CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$HADOOP_HOME/bin:$JAVA_HOME/bin:$JAVA_HOME/jre/bin
```

* 创建hadoop所需目录
`mkdir /home/test/app/hadoop-2.7.4/{name,data,tmp}`

# hadoop配置文件修改[主节点]
* hadoop-env.sh和yarn-env.sh  

```
export JAVA_HOME=/home/test/app/jdk1.8.0_121
```

* slaves  

```
Bgdata03
Bgdata04
Bgdata05
```

* hdfs-site.xml  

```xml
<configuration>
<property>
  <name>dfs.namenode.name.dir</name>
  <value>file:/home/test/app/hadoop-2.7.4/name</value>
</property>
<property>
  <name>dfs.datanode.data.dir</name>
  <value>file:/home/test/app/hadoop-2.7.4/data</value>
</property>
<property>
  <name>dfs.replication</name>
  <value>3</value>
</property>
<property>
  <name>dfs.webhdfs.enabled</name>
  <value>true</value>
</property>
</configuration>
```

* core-site.xml  

```xml
<configuration>
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://Bgdata03:9000</value>
</property>
<property>
  <name>hadoop.tmp.dir</name>
  <value>file:/home/test/app/hadoop-2.7.4/tmp</value>
</property>
</configuration>
```

* mapred-site.xml  

```
<configuration>
<property>
  <name>mapreduce.framework.name</name>
  <value>yarn</value>
</property>
<property>
  <name>mapreduce.jobhistory.address</name>
  <value>Bgdata03:10020</value>
</property>
<property>
  <name>mapreduce.jobhistory.webapp.address</name>
  <value>Bgdata03:19888</value>
</property>
</configuration>
```

* yarn-site.xml  

```xml
<configuration>
<property>  
  <name>yarn.nodemanager.aux-services</name>  
  <value>mapreduce_shuffle</value>  
</property>  
<property>  
  <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>  
  <value>org.apache.hadoop.mapred.ShuffleHandler</value>  
</property>  
<property>  
  <name>yarn.resourcemanager.address</name>  
  <value>Bgdata03:8032</value>  
</property>  
<property>  
  <name>yarn.resourcemanager.scheduler.address</name>  
  <value>Bgdata03:8030</value>  
</property>  
<property>  
  <name>yarn.resourcemanager.resource-tracker.address</name>  
  <value>Bgdata03:8031</value>  
</property>  
<property>  
  <name>yarn.resourcemanager.admin.address</name>  
  <value>Bgdata03:8033</value>  
</property>  
<property>  
  <name>yarn.resourcemanager.webapp.address</name>  
  <value>Bgdata03:8088</value>  
</property>  
</configuration>
```

# 复制
```shell
rsync -azvP --delete -e ssh hadoop-2.7.4/ Bgdata04:~/app/hadoop-2.7.4/
rsync -azvP --delete -e ssh hadoop-2.7.4/ Bgdata05:~/app/hadoop-2.7.4/
```
# 格式化hdfs[主节点]
hdfs namenode -format  
# 启动hadoop[主节点]  
```shell
cd hadoop-2.7.4/
./sbin/start-dfs.sh 
./sbin/start-yarn.sh
```

# 部署信息查询
* 命令行方式[jps]

namenode:  

>9921 NameNode
10518 Jps
10108 SecondaryNameNode
10253 ResourceManager

datanode:  

>18689 Jps
18475 DataNode
18539 NodeManager  

* 浏览器方式  
  * 查看yarn页面 <http://192.168.99.101:8088>
  * 查看节点信息统计 <http://192.168.99.101:50070>  

# 功能测试
* 本地建立文件  

```
mkdir input
cd input/
echo "hello hadoop" > test2.txt
echo "hello world" > test1.txt
```

* hdfs建立目录  

```
hadoop fs -put ./input ./in
hadoop fs -ls ./in
```

* 使用内置demo测试  

```
hadoop jar /home/hadoop/hadoop-2.7.3/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.3.jar wordcount in out
hadoop fs -ls ./out
hadoop fs -cat ./out/part-r-00000
```

输出如下:  
>hadoop  1
hello   2
world   1

# 安装参考
* [VM下CentOS7x86-64bit+JDK1.8+hadoop2.7.2安装部署][1]
* [分布式安装hadoop][2]

[1]:http://blog.csdn.net/wzh_lucky/article/details/50761923
[2]:http://blog.csdn.net/clark_xu/article/details/69668618
[3]:../Ops/ssh_conn_all.html
[4]: http://mirrors.aliyun.com/apache/hadoop/common/hadoop-2.7.4/hadoop-2.7.4.tar.gz
[5]: http://47.93.30.16/jdk-8u121-linux-x64.tar.gz
