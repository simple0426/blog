# 软件下载
* [spark][1]
* [scala][2]

# 软件安装
> 由于spark依赖于scala，所以需要优先安装scala

* 解压压缩包scala和spark
* 配置环境变量

```
export SPARK_HOME=/home/test/app/spark-2.1.1-bin-hadoop2.7
export SCALA_HOME=/home/test/app/scala-2.12.3
export PATH=$PATH:$SPARK_HOME/bin:$SCALA_HOME/bin
```

# 配置文件修改
* spark-env.sh

```
export SCALA_HOME=/home/test/app/scala-2.12.3
export JAVA_HOME=/home/test/app/jdk1.8.0_121
export HADOOP_HOME=/home/test/app/hadoop-2.7.4
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export SPARK_HOME=/home/test/app/spark-2.1.1-bin-hadoop2.7
export SPARK_MASTER_IP=Bgdata04
export SPARK_EXECUTOR_MEMORY=1G
```

* slaves

```
Bgdata03
Bgdata04
Bgdata05
```

# 复制
```
rsync -azvP -e ssh spark-2.1.1-bin-hadoop2.7 Bgdata05:~/app/
rsync -azvP -e ssh spark-2.1.1-bin-hadoop2.7 Bgdata04:~/app/
```
# 启动：
* `./sbin/start-all.sh`
* 状态查看：<http://127.0.0.1:8080/>

# 测试
## 调用Spark自带的计算圆周率的Demo
`./bin/spark-submit --class  org.apache.spark.examples.SparkPi  --master local   examples/jars/spark-examples_2.11-2.1.1.jar  `
## 使用spark-shell

```
val file=sc.textFile("hdfs://Bgdata03:9000/user/hive/warehouse/xp/test.txt")
val rdd = file.flatMap(line => line.split(" ")).map(word => (word,1)).reduceByKey(_+_)
rdd.collect()
rdd.foreach(println)
# 退出
:quit
```

# 安装参考
* [Hadoop2.7.3+Spark2.1.0][3]
* [Linux安装Spark集群][4]

[1]:http://mirrors.aliyun.com/apache/spark/spark-2.1.1/spark-2.1.1-bin-hadoop2.7.tgz
[2]: https://downloads.lightbend.com/scala/2.12.3/scala-2.12.3.tgz
[3]: http://www.cnblogs.com/purstar/p/6293605.html
[4]: http://blog.csdn.net/pucao_cug/article/details/72353701
