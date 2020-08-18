---
title: jenkins安装与配置
date: 2020-02-10 15:12:23
tags:
    - jenkins
categories: ['CICD']
---

# 安装
> 需要提前配置java8运行环境

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
  -e JAVA_OPTS="-Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true" \
  --restart=always \
  jenkins/jenkins:lts
```
* 参数详解：
    - --name：设置容器名称
    - -u：以root启动，防止出现权限问题
    - -d：后台运行
    - -p 7005:8080：映射服务端口【宿主机7005-》容器8080】
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
    sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && \
    sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
    ```
- 重启jenkins：`docker restart jenkins`

# 邮件设置

> Mailer插件

- 系统管理员帐户
- SMTP服务器户
- SMTP认证
- 用户名
- 密码
- 发送测试【如果设置的邮件为163，则收件人和发件人一样；否则会显示为垃圾邮件不能发送】
- 测试用户邮箱地址

# 节点管理

> 一般使用ssh连接并控制slave节点，需要[SSH Build Agents](https://plugins.jenkins.io/ssh-slaves)插件

- 远程工作目录
- 设置启动方式（ssh）
- 节点属性
  - 环境变量：如JAVA_HOME、MAVEN_HOME
  - tools位置：参见工具[使用注意](#使用注意)

# 全局工具

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
* 指定install automatically，从java.sun.com安装【需要oracle账号】
* 指定install automatically，从指定的url处下载\*.zip/\*.tar.gz文件；__注意__：必须指定解压子目录，且子目录名称为解压的原始目录，如
  * URL：https://jre.injdk.cn/oracle/8/jdk-8u251-linux-x64.tar.gz
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

# 集成docker

* 插件
    - Docker Commons Plugin：给各种docker插件提供基础的API
    - docker-build-step：在job中执行docker命令
* 配置(docker-build-step插件)：系统配置--》Docker Builder--》unix:///var/run/docker.sock【确保jenkinis用户对此socket有读权限】
* 使用方式-tools：通过在全局工具配置中定义特定版本的docker
```
pipeline {
  agent any
  tools {
    'org.jenkinsci.plugins.docker.commons.tools.DockerTool' 'docker1903'
  }
  stages {
    stage('foo') {
      steps {
        sh "docker version"
      }
    }
  }
}
```
* 使用方式-agent：加载PATH变量下默认的docker
```
pipeline{
    agent{
        docker{
            image 'maven:3.6.3-jdk-8'
            label 'agent01'
        }
    }
    stages{
        stage('Build'){
            steps{
                sh 'echo jdk version...'
                sh 'java -version'
            }
        }
        stage('Test'){
            steps{
                sh 'echo maven version...'
                sh 'mvn --version'
            }
        }
        stage('Deploy'){
            steps{
                sh 'echo Deploy stage...'
            }
        }
    }
}
```
* 并行使用多个docker镜像(parallel)
```
stage('Build'){
    parallel{
        stage('Front End Build: Angular'){
            agent{
                docker{
                    image 'liumiaocn/angular:8.3.8'
                }
            }
            steps{
                sh 'echo Front End Build stage...'
                sh 'ng --version'
            }
        }
        stage('Back End build: Marven'){
            agent{
                docker{
                    image 'maven:3.6.3-jdk-8'
                }
            }
            steps{
                sh 'echo Back End Build stage ...'
                sh 'mvn --version'
            }
        }
    }
}
```
* dockerfile中定义构建环境
    - 默认，需要在多分支pipeline或SCM-pipeline中使用
    - 为了手动测试，可以在jenkins宿主机的绝对目录定义dockerfile
```
agent{
    dockerfile{
        filename 'Dockerfile'
        dir '/tmp'
        label 'agent01'
    }
}
```
