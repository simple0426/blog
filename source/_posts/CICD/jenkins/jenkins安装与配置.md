---
title: jenkins安装与配置
date: 2020-02-10 15:12:23
tags:
    - jenkins
categories: ['CICD']
---

# 安装
##  配置java运行环境

需要注意jenkins与java版本的匹配性【由于节点管理的功能，会有主节点的jenkins产生一个jar包在node上运行，如果node上的默认java版本过低，则jar运行报错】

https://www.jenkins.io/doc/book/platform-information/support-policy-java/

```
cd /usr/local/
tar xzf jdk*.tar.gz
tar xzf apache-maven*.tar.gz
ln -s apache-maven* maven
ln -s jdk* jdk
vim /etc/profile
MAVEN_HOME=/usr/local/maven
JAVA_HOME=/usr/local/jdk
export PATH=$PATH:$MAVEN_HOME/bin:$JAVA_HOME/bin
source /etc/profile
```

## 虚拟机方式

* centos安装
```
wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat/jenkins.repo
rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
yum install jenkins -y
```
* ubuntu安装
```
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```
* 源码包安装
    - 软件源：`https://mirrors.tuna.tsinghua.edu.cn/jenkins/` `debian-stable|redhat-stable`
    - deb包安装失败
        + sudo apt-get update # 更新
        + sudo apt-get -f install # 解决依赖关系
        + sudo dpkg -i xxx.deb # 重新安装
    - 安装错误
        + 错误：ERROR: No Java executable found in current PATH: /bin:/usr/bin:/sbin:/usr/sbin
        + 解决：ln -s /usr/local/jdk1.8.0_241/bin/java /usr/bin/

## docker方式

* 启动参数
```
docker run --name jenkins \
  -u root -d \
  -p 80:8080 -p 50000:50000 \
  -v /data/jenkins/:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/bin/docker:/usr/bin/docker \
  -v /usr/local/maven:/usr/local/maven \
  -v /usr/local/jdk:/usr/local/jdk \
  -v /etc/localtime:/etc/localtime \
  -e TZ=Asia/Shanghai \
  -e JAVA_OPTS="-Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true" \
  --restart=always \
  jenkins/jenkins:lts
```
* 参数详解：
    - --name：设置容器名称
    - -u：以root启动，防止出现权限问题
    - -d：后台运行
    - -p 80:8080：映射服务端口【宿主机80-》容器8080】
    - -p 50000:50000：agent连接master的端口
    - -v /data/jenkins/:/var/jenkins_home：jenkins主目录持久化存储
    - -v /var/run/docker.sock:/var/run/docker.sock：确保jenkins容器内可以操作宿主机的docker
    - -v /usr/local/maven:/usr/local/maven：挂载宿主机maven到jenkins容器
    - -v /usr/local/jdk:/usr/local/jdk：挂载宿主机jdk到容器
    - -v /etc/localtime:/etc/localtime：设置容器时间
    - -e JAVA_OPTS="-Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true"：关闭csrf保护
* csrf设置：由于jenkins2.0版本默认开启CSRF且不能关闭，所以需要在启动jenkins服务时关闭csrf
    - [官方说明](https://www.jenkins.io/doc/upgrade-guide/2.222/#always-enabled-csrf-protection)
    - [网友设置](https://www.cnblogs.com/kazihuo/p/12937071.html)

# 访问和认证
* 访问地址：http://JENKINS_URL:8080
    - 初始化密码：/var/jenkins_home/secrets/initialAdminPassword

# 使用国内镜像

- 修改插件更新中心URL：`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`

- 修改json配置【/var/jenkins_home/updates/default.json】

  > 每次插件信息(default.json)更新都会覆盖为官方地址，所以每次default.json更新都需要做如下修改

    ```
  sed -i 's/https:\/\/updates.jenkins.io\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && \
   sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
    ```
- 重启jenkins：`docker restart jenkins`

# pipeline

pipeline是jenkins中的工作方式，类似linux中的脚本；

pipeline可以将运维流程以脚本方式进行标准化、自动化

需要安装插件：pipeline、pipeline stage view、git

# 邮件设置

> Mailer插件

- 系统管理员帐户
- SMTP服务器
- SMTP认证
  - 用户名
  - 密码
- （临时）收件人邮箱

# 节点管理

> 一般使用ssh连接并控制slave节点，需要[SSH Build Agents](https://plugins.jenkins.io/ssh-slaves)插件

- 远程工作目录
- 设置启动方式（ssh）
  - Host Key Verification Strategy：None
- 节点属性
  - 环境变量：如JAVA_HOME、MAVEN_HOME
  - tools位置：参见工具[使用注意](#使用注意)

# 全局工具

使用工具后，jenkins可以并安装并配置maven、jdk等工具，并把相关的环境变量MAVEN_HOME、JAVA_HOME放在执行环境的PATH变量下，这样就可以直接使用相关的命令java、mvn

## maven

> Maven Configuration处本意为设置全局的maven settings.xml文件，经过多次试验均无效；所以Maven Configuration保持默认设置；

maven有多种安装方式

* 在宿主机安装maven后，在Maven installations处指定MAVEN_HOME
* 指定install automatically，从Apache官方自动安装maven
* 指定install automatically，从指定的url处下载\*.zip/\*.tar.gz文件；__注意__：必须指定解压子目录，且子目录名称为解压的原始目录，如
  * URL:https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
  * 子目录：apache-maven-3.6.3

## jdk

> 与maven安装类似

jdk也有多种安装方式

* 在宿主机安装jdk后，在JDK installations处指定JAVA_HOME
* 指定install automatically，从指定的url处下载\*.zip/\*.tar.gz文件；__注意__：必须指定解压子目录，且子目录名称为解压的原始目录，如
  * URL：https://d6.injdk.cn/oracle/8/jdk-8u251-linux-x64.tar.gz 
  * 子目录：jdk1.8.0_251

## 使用注意

* node节点tools使用：

  > 全局工具设置只在master节点有效，node节点需要相关工具时需要在node节点单独安装；node节点工具使用设置如下：

  ```
  节点管理==》节点属性==》工具位置：
  * 名称：master和node都可以使用的工具名称
  * Home：maven、jdk等工具的根目录，和JAVA_HOME、MAVEN_HOME类似
    - jdk设置：如/usr/local/jdk【JAVA_HOME】
    - maven设置：如/usr/local/maven【MAVEN_HOME】
  
  由于tools设置的工具，尤其是自动化安装操作只在master上有效，所以为了统一管理，建议master和node上统一
  使用本地安装方式，并保持目录位置一致
  ```

* maven仓库设置：直接修改安装目录下的conf/settings.xml

  ```
     <mirror>
       <id>nexus-aliyun</id>
       <mirrorOf>central</mirrorOf>
       <name>Nexus aliyun</name>
       <url>http://maven.aliyun.com/nexus/content/groups/public</url>
     </mirror>
  ```

* 容器方式挂载的jdk、maven

  * 可以在tools中设置JAVA_HOME/MAVEN_HOME，然后以tools方式使用maven、jdk
  * 也可以在pipeline中以shell方式设置JAVA_HOME/MAVEN_HOME，然后直接使用maven、jdk

## 使用范例

```
pipeline {
    agent any
    tools {
      jdk 'jdk-251'
      maven 'mvn-3.5'
    }
    stages {
        stage('java') {
            steps {
                sh 'java -version'
            }
        }
        stage('maven') {
            steps {
                sh 'mvn -version'
            }
        }
    }
}
```
