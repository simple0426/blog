---
title: k8s实施-微服务与SpringCloud容器化
date: 2020-09-16 21:36:33
tags:
  - 微服务
  - SpringCloud
  - APM
  - Pinpoint
categories: ['kubernetes']
---


# 运维角度看微服务

## 单体应用vs微服务

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/microservice.png)

**单体应用**：将全部的功能在一个应用中实现，优缺点如下：

* 优点：易于部署和测试
* 缺点：
  * 代码量膨胀，难以维护【服务之间耦合性太高，单个应用变动可能引起其他服务不可用】
  * 构建和部署成本大【编译时间随代码量增加而增加、单个功能更新需要重新部署整个应用】
  * 扩展困难

**微服务**：是一种设计风格，将不同的功能拆分到不同的应用中实现，不同的子应用通过REST api进行通信；必备组件包括：

* 注册中心：微服务的服务发现、负载均衡实现
* API Gateway：微服务的访问入口

## 微服务特点

* 服务组件化：每个服务独立开发，减少应用间的依赖关系，有效避免了一个服务的修改造成其他服务不可用。
* 技术栈灵活：约定通信方式后(RPC、RESTapi)，不同的功能可以使用不同的语言开发
* 独立数据：每个微服务可以有独立的基本组件，例如数据库、缓存等。
* 独立部署：每个微服务独立部署，加快构建和部署速度
* 扩展性强：每个微服务可以部署多个实例，并且有负载均衡能力

## 微服务难点和不足

* 架构复杂性和沟通成本：每个功能的完成，都需要多个组件的系统工作；组件之间的协同工作需要整体的架构设计、接口和文档的标准化等工作
* 数据一致性：多个组件间事务性操作(订单、支付、库存等)需要确保数据的一致性
* 运维成本：随着组件的增多，服务的部署、配置、监控、日志等微服务的治理工作量也随之增多，要求的自动化程度、专业技能也随之增强

## java微服务框架

spring boot：微服务开发框架

* dubbo：微服务治理框架
* spring cloud：基于spring boot的完整微服务解决方案

# K8S中部署微服务

## 架构图

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/microservice-1.png)

## 微服务架构理解

* 微服务间如何通信？REST API，RPC，MQ
* 微服务如何发现彼此？注册中心
* 组件之间怎么个调用关系？
* 哪个服务作为整个网站入口？前后端分离
* 哪些微服务需要提供外部访问入口？前端、微服务网关
* 微服务怎么部署？更新？扩容？
  * 部署：kubectl apply、helm install【jenkins】
  * 更新：kubectl set image
  * 扩容：kubectl scale

* 区分有状态应用与无状态应用：有状态一般部署在虚拟机、无状态部署在k8s中
  * 应用服务：订单、商品、库存、支付
  * 数据库/存储：数据库、redis、MQ、分布式存储

## 为什么要用注册中心

微服务太多面临的问题：

* 怎么记录一个微服务多个副本接口地址？
* 怎么实现一个微服务多个副本负载均衡？
* 怎么判断一个微服务副本是否可用？

主流注册中心：Eureka(spring cloud)，Nacos(alibaba)，Consul

## 容器交付流程

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/microservice-cicd.png)

## 在K8s部署项目流程

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/microservice-k8s.png)

# 微服务范例-Spring Cloud

具体步骤：

前置条件：在K8s中部署MySQL数据库、编译主机安装maven环境、启动harbor服务
第一步：熟悉Spring Cloud微服务项目
第二步：源代码编译构建
第三步：构建项目镜像并推送到镜像仓库
第四步：K8s服务编排
第五步：在K8s中部署注册中心(Eureka集群)
第六步：部署微服务业务程序（product、stock、order）
第七步：部署微服务网关服务（gateway）
第八步：部署微服务前端（portal）
第九步：微服务对外发布
第十步：微服务升级与扩容

**自动化构建说明**：使用项目中的脚本docker_build.sh可以完成2~9部操作

## 前置条件

* helm安装mysql

  ```
  helm repo add microsoft http://mirror.azure.cn/kubernetes/charts/
  helm repo update
  helm install db --set persistence.storageClass="example-nfs" microsoft/mysql
  helm status db
  ```

* maven环境设置【编译项目的主机】

  ```
  1. 安装maven
  yum install java-1.8.0-openjdk maven
  
  2. 设置maven源
  <mirror>
   <id>nexus-aliyun</id>
   <mirrorOf>central</mirrorOf>
   <name>Nexus aliyun</name>
   <url>http://maven.aliyun.com/nexus/content/groups/public</url>
  </mirror>
  ```

## 熟悉Spring Cloud微服务项目

* 项目地址：https://github.com/simple0426/simple-microservice.git

* 代码分支说明：
  * dev3 增加K8S资源编排【k8s/*.yaml】
  * dev4 增加微服务链路监控【*/pinpoint-agent】
  * master 最终上线
* 项目说明

```
1. k8s部署使用dev3分支
2. 导入db目录下数据库文件到自己的MySQL服务器，操作如下
3. 修改连接数据库配置（xxx-service/src/main/resources/application-fat.yml）
4. 修改前端页面连接网关地址（portal-service/src/main/resources/static/js/productList.js和orderList.js）【默认：gateway.ctnrs.com】
5. 修改k8s/docker_build.sh中的harbor信息
```

* 连接mysql并导入sql文件

```
yum install -y mysql
MYSQL_HOST=$(kubectl get svc|awk '/db-mysql/{print $3}')
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default db-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)
mysql -h$MYSQL_HOST -P3306 -uroot -p${MYSQL_ROOT_PASSWORD} < order.sql 
mysql -h$MYSQL_HOST -P3306 -uroot -p${MYSQL_ROOT_PASSWORD} < product.sql 
mysql -h$MYSQL_HOST -P3306 -uroot -p${MYSQL_ROOT_PASSWORD} < order.sql
```

## 源代码编译构建

编译【由于项目bug，不能针对单个微服务打包，所以需要在项目根目录下操作】

```
mvn clean package -Dmaven.test.skip=true 
```

## 构建项目镜像并推送到镜像仓库

* Dockerfile

```
FROM java:8-jdk-alpine
LABEL maintainer lizhenliang/www.ctnrs.com
RUN  ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
COPY ./target/eureka-service.jar ./
EXPOSE 8888
CMD java -jar -Deureka.instance.hostname=${MY_POD_NAME}.eureka.ms /eureka-service.jar
```

* build && push

```
docker build -t ${image_name} .
docker push ${image_name} 
```

## K8s服务编排

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/microservice-k8s-1.png)

* 前端portal、微服务网关gateway需要使用ingress对外暴露，以方便用户访问
* 业务服务(product、stock、order)只需使用deployment部署pod，通过注册中心(Eureka)进行服务发现和负载均衡
* 注册中心使用statefulset进行部署

## 部署Erureka集群

* Erureka集群实现

  ```
  1. pod设置变量【Deployment】
  env:
    - name: MY_POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
  
  2. 程序启动时引用变量【Dockerfile】
  CMD java -jar -Deureka.instance.hostname=${MY_POD_NAME}.eureka.ms /eureka-service.jar
  
  3. 程序中设置集群信息【eureka-service/src/main/resources/application-fat.yml】
  defaultZone: http://eureka-0.eureka.ms:${server.port}/eureka/,http://eureka-1.eureka.ms:${server.port}/eureka/,http://eureka-2.eureka.m
  s:${server.port}/eureka/
  ```

* Erureka web访问

  ```
  1. 查看访问地址
  kubectl get ingress -n ms
  
  2. windows主机hosts设置
  192.168.31.202 eureka.ctnrs.com
  
  3. web浏览器访问
  http://eureka.ctnrs.com/
  ```

## 在K8s中部署微服务

>eureka客户端设置【prefer-ip-address: true】使微服务以pod的ip注册

* 部署业务程序（product、stock、order）
* 部署网关（gateway）
* 部署前端（portal）

## 微服务升级与扩容

* 微服务升级：对要升级的微服务进行上述步骤打包镜像:版本，替代运行的镜像
* 微服务扩容：对Pod扩容副本数

# K8S部署实践

## JVM内存限制

> 参考1：https://blog.csdn.net/gBbQRglVIr3dYi82/article/details/82754927
>
> 参考2：https://www.oracle.com/java/technologies/javase/8u191-relnotes.html
>
> 参考3：https://www.cnblogs.com/xiaoqi/p/container-jvm.html

```
1. java 5/6/7/8u131-
设置-Xmx

2. java 8u131+和java9+
设置-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap

3.java 8u191+和java10+
UseContainerSupport默认开启，可以感知容器内存和cpu限制；同时设置-XX:MaxRAMPercentage=75.0，这样为其他进程（debug、监控）留下足够的内存空间
```

## 滚动更新-高可用

滚动更新是默认发布策略，当配置健康检查时，滚动更新会根据Probe状态来决定是否继续更新以及是否允许接
入流量，这样在整个滚动更新过程中可保持始终会有可用的Pod存在，达到平滑升级。

```
readinessProbe:
  tcpSocket:
    port: 9999
  initialDelaySeconds: 60
  periodSeconds: 10
livenessProbe:
  tcpSocket:
    port: 9999
  initialDelaySeconds: 60
  periodSeconds: 10
```

## 滚动更新-pod删除

滚动更新触发，Pod在删除过程中，有些节点kube-proxy还没来得及同步iptables规则，从而部分流量请求到Terminating的Pod上，导致请求出错。
解决办法：配置preStop回调，在容器终止前优雅暂停5秒，给kube-proxy多预留一点时间。

```
lifecycle:
  preStop:
    exec:
      command:
      - sh
      - -c
      - "sleep 5"
```

# 微服务链路监控-APM

## 全链路监控是什么

随着微服务架构的流行，服务按照不同的维度进行拆分，一次请求往往需要涉及到多个服务。这些服务可能用不同编程语言开发，不同团队开发，可能部署很多副本。因此，就需要一些可以帮助理解系统行为、用于分析性能问题的工具，以便发生故障的时候，能够快速定位和解决问题。全链路监控组件就在这样的问题背景下产生了。

全链路性能监控 从整体维度到局部维度展示各项指标，将跨应用的所有调用链性能信息集中展现，可方便度量整体和局部性能，并且方便找到故障产生的源头，生产上可极大缩短故障排除时间。

## 全链路监控解决的问题

* 请求链路追踪：通过分析服务调用关系，绘制运行时拓扑信息，可视化展示
* 调用情况衡量：各个调用环节的性能分析，例如吞吐量、响应时间、错误次数
* 容量规划参考：扩容/缩容、服务降级、流量控制
* 运行情况反馈：告警，通过调用链结合业务日志快速定位错误信息

## 全链路监控组件选择

全链路监控系统有很多，应从这几方面选择：

* 探针的性能消耗
  APM组件服务的影响应该做到足够小，数据分析要快，性能占用小。
* 代码的侵入性
  即也作为业务组件，应当尽可能少入侵或者无入侵其他业务系统，
  对于使用方透明，减少开发人员的负担。
* 监控维度
  分析的维度尽可能多。
* 可扩展性
  一个优秀的调用跟踪系统必须支持分布式部署，具备良好的可扩展
  性【服务端分布式部署】。能够支持的组件越多当然越好。

常用APM组件：zipkin、skywalking、pinpoint

## Pinpoint介绍

Pinpoint是一个APM（应用程序性能管理）工具，适用于用Java/PHP编写的大型分布式系统。

可以监控的信息如下：

* 服务器地图（ServerMap）通过可视化分布式系统的模块和他们之间的相互联系来理解系统拓扑。点击某个节点会 展示这个模块的详情，比如它当前的状态和请求数量。
* 实时活动线程图 （Realtime Active Thread Chart） ：实时监控应用内部的活动线程。
* 请求/响应分布图（ Request/Response Scatter Chart ） ：长期可视化请求数量和应答模式来定位潜在问题。通过在图表上拉拽可以选择请求查看 更多的详细信息。
* 调用栈（ CallStack ）：在分布式环境中为每个调用生成代码级别的可视图，在单个视图中定位瓶颈和失败点。
* 检查器（ Inspector ） ：查看应用上的其他详细信息，比如CPU使用率，内存/垃圾回收，TPS，和JVM参数。

## Pinpoint服务端部署

* docker方式部署：https://github.com/naver/pinpoint-docker

  >启动时会在本地构建镜像(docker-compose build)，构建过程中会从github拉取各组件的tar包，
  >
  >所以构建过程很慢

  ```
  git clone https://github.com/naver/pinpoint-docker.git
  cd pinpoint-docker
  docker-compose pull && docker-compose up -d
  ```

* 部署完成后包含的组件

  ```
  pinpoint-agent            # pinpoint agent组件（为应用程序范例提供agent程序） 
  pinpoint-collector        # 收集agent的数据                   
  pinpoint-docker_zoo1_1    
  pinpoint-docker_zoo2_1    
  pinpoint-docker_zoo3_1    
  pinpoint-flink-jobmanager 
  pinpoint-flink-taskmanager                          
  pinpoint-hbase             # 保存指标数据                            
  pinpoint-mysql             # 保存web页面数据
  pinpoint-quickstart        # 应用程序范例，通过共享存储挂载agent组件
  pinpoint-web               # 服务端web界面
  ```

* 访问

  ```
  * 应用程序范例：http://xxxx:8000
  * pinpoint服务端web界面：http://xxxx:8079
  ```

## [Pinpoint Agent部署](naver.github.io/pinpoint/installation.html#5-pinpoint-agent)

* pinpoint agent程序供给

  * 程序获取：

    * pinpoint web界面右侧【设置】--》installation--》Download Link
    * 实际地址：`https://github.com/naver/pinpoint/releases/download/v2.1.0/pinpoint-agent-2.1.0.tar.gz`

  * 程序设置【v2.1.0】

    ```
    * pinpoint-root.config
    pinpoint.profiler.profiles.active=release # 设置加载的环境
    # 其他为全局配置，环境级别的配置会覆盖同名条目信息
    
    * profiles/release/pinpoint.config        # release环境设置
    profiler.transport.module=GRPC            # agent与server以grpc方式通信
    profiler.transport.grpc.collector.ip=192.168.31.250      # server地址
    ```

  * agent程序位置

    ```
    1. 通过initContainer方式将pinpoint agent程序复制到共享目录供应用程序使用
    2. 启动一个job或pod，将pinpoint agent程序挂载到集群存储中，其他pod挂载这个存储
    3. 制作镜像Dockerfile时，将pinpoint agent程序复制到镜像中
    ```

* 应用程序启动时加载pinpoint agent

```
tomcat设置：
#catalina.sh
CATALINA_OPTS="$CATALINA_OPTS -javaagent:$AGENT_PATH/pinpoint-bootstrap-$VERSION.jar"
CATALINA_OPTS="$CATALINA_OPTS -Dpinpoint.agentId=$AGENT_ID"
CATALINA_OPTS="$CATALINA_OPTS -Dpinpoint.applicationName=$APPLICATION_NAME"

jar设置：
java -jar -javaagent:$AGENT_PATH/pinpoint-bootstrap-$VERSION.jar -Dpinpoint.agentId=$AGENT_ID
-Dpinpoint.applicationName=$APPLICATION_NAME xxx.jar
```


